# Approfondissement — complétion, processus et contrat machine

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Le cours définit les arguments, les statuts et les événements d'une commande.
Ce complément étudie ce qui se passe avant son exécution et à la frontière du processus PHP.
Console et Process sont déjà installés ; aucun service externe n'est nécessaire.
Les API utilisées appartiennent au socle Symfony 8.0 et sont présentes dans le dépôt 8.1.8.
Les exemples sont des fragments d'observation, pas les commandes demandées dans les exercices.

## 1. Une suggestion n'est pas une validation

La complétion reçoit une saisie inachevée, par exemple une option sans valeur complète.
Console fournit un `CompletionInput` capable de représenter cet état temporairement invalide.
La commande propose des valeurs ; le shell choisit celles à afficher selon son comportement.
Avec un attribut `Option`, `suggestedValues` peut être une liste ou un fournisseur de suggestions.
Une liste courte et stable évite toute lecture coûteuse pendant la frappe.

```php
use Symfony\Component\Console\Attribute\Option;

// Fragment de paramètre dans une commande invocable :
#[Option(suggestedValues: ['compact', 'detailed'])]
string $view = 'compact',
```

Ces suggestions ne refusent pas une valeur saisie manuellement comme `--view=inconnu`.
La commande doit encore vérifier son contrat avant d'appeler le service métier.
La complétion ne doit ni lancer l'analyse CSV ni écrire le rapport.
Elle peut être sollicitée plusieurs fois sans que l'utilisateur n'exécute ensuite la commande.
Dans l'API classique, `complete()` reçoit l'entrée et un `CompletionSuggestions`.
`mustSuggestOptionValuesFor()` distingue les valeurs de l'option des noms d'options.
Si tu redéfinis `complete()`, préserve les suggestions existantes quand elles sont pertinentes.

Le tester dédié permet d'appeler ce mécanisme sans installer un script dans ton shell :

```php
use Symfony\Component\Console\Tester\CommandCompletionTester;

// $command est une commande déjà obtenue depuis l'application Console.
$completion = new CommandCompletionTester($command);
$values = $completion->complete(['--view', '']);
```

Le dernier élément vide représente une valeur encore à saisir.
Ce tester retourne les suggestions produites ; ne lui attribue pas tout le filtrage du shell.
Teste la liste pertinente et l'absence d'appel au traitement métier.
Teste séparément une exécution réelle avec une valeur hors liste.
La [documentation des entrées Console 8.0](https://symfony.com/doc/8.0/console/input.html#adding-argument-option-value-completion) décrit ce protocole.

## 2. Du tester au vrai processus

`CommandTester` travaille dans le processus PHPUnit et appelle directement une commande.
`ApplicationTester` ajoute le parsing et les événements orchestrés par l'application.
Un sous-processus PHP ajoute encore le démarrage Runtime, l'environnement et les vrais flux système.
Ces niveaux répondent à des questions différentes ; le dernier ne remplace pas tous les premiers.
Le composant Process accepte un tableau d'arguments : chaque élément conserve son identité.
Un chemin contenant des espaces ne demande alors aucun assemblage de guillemets à la main.

```php
use Symfony\Component\Process\Process;

// $projectDir désigne la racine du dépôt.
$process = new Process([PHP_BINARY, 'bin/console', 'about', '--env=test'], $projectDir);
$status = $process->run();
$stdout = $process->getOutput();
$stderr = $process->getErrorOutput();
```

`run()` conserve un statut d'échec pour l'inspection ; `mustRun()` lève une exception en cas d'échec.
Pour vérifier un statut 1 ou 2 attendu, `run()` rend l'intention du test plus directe.
Le tableau d'arguments évite une interprétation par le shell ; Console analyse encore les options.
Ainsi, un nom commençant par `-` peut toujours nécessiter le séparateur Console `--`.
Le répertoire de travail doit être explicite pour que `bin/console` et les fichiers relatifs soient trouvés.
Les doubles posés dans le conteneur PHPUnit ne sont pas transmis au nouveau processus.
Le sous-processus doit donc utiliser une configuration de test et des fixtures qu'il peut réellement lire.
Voir [Process 8.0](https://symfony.com/doc/8.0/components/process.html) pour l'environnement et les délais.

## 3. Définir les deux flux comme une interface

Une sortie machine a besoin d'un format, mais aussi d'un emplacement stable.
Un JSON correct précédé d'un message de démarrage n'est plus un document JSON unique.
Réserve la sortie standard au résultat lorsque l'option de format machine est choisie.
Les diagnostics vont sur la sortie d'erreur ; le code de sortie garde sa fonction de décision.
Fixe explicitement ce que produit la commande lorsqu'elle ne peut même pas ouvrir son entrée.
Par exemple, « aucun rapport sur stdout et un diagnostic sur stderr » est un contrat testable.
Un rejet de ligne peut, lui, produire un rapport complet accompagné d'un statut non nul.

| Observation recherchée | Outil adapté | Cas négatif indispensable |
|---|---|---|
| Valeurs proposées avant exécution | `CommandCompletionTester` | Aucun appel métier |
| Résultat et statut d'une commande | `CommandTester` | Paramètre métier invalide |
| Effet d'un listener Console | `ApplicationTester` | Listener absent au niveau direct |
| Flux et démarrage PHP réels | `Process` | Fichier absent, statut conservé |
| Argument avec espaces | Tableau d'arguments Process | Pas de découpage du chemin |

## Observer dans l'application

Après création des commandes de l'atelier :

```bash
php bin/console help app:contacts:inspect
php bin/console completion --help
php bin/console debug:container --tag=console.command --env=test
php bin/phpunit tests/Console
```

Compare un test direct puis un test de sous-processus sur la même fixture temporaire.
Dans le second, relève séparément statut, stdout et stderr avant d'expliquer le résultat.
Nettoie les fichiers même quand une assertion échoue ; ne réutilise pas un rapport d'un ancien test.
Le profiler n'est pas nécessaire pour observer cette frontière.
Sources locales : `console/Tester/CommandCompletionTester.php`, `console/Input/InputOption.php` et `process/Process.php` sous `vendor/symfony/`.
Le [tester de complétion en 8.0.0](https://github.com/symfony/console/blob/v8.0.0/Tester/CommandCompletionTester.php) confirme le traitement des jetons.
L'API `CommandTester::run()` introduite en 8.1 reste hors de ce complément : conserver `execute()` pour le socle.
