# Module 13 — Console : contrats d'entrée et automatisation

[Parcours](../README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Construire une commande utilisable au terminal comme en CI : entrées explicites, sorties stables et codes de retour utiles.
Comprendre le cycle d'exécution, les interactions et la différence entre tester une commande et tester l'application Console.
Prérequis : [DI](../03-dependency-injection/README.md), [Validator](../12-validator/README.md), PHP et exceptions.
Fil rouge : analyser un CSV de contacts et produire un rapport, avec simulation `--dry-run`, sans base de données.
Le service retourne un résultat en mémoire ; seul le rapport explicitement demandé peut être écrit dans un fichier temporaire.
Code futur : `src/Console/`, tests : `tests/Console/`.
Symfony **8.1.8**, PHP **8.4.1** ; les exemples principaux restent compatibles avec le socle **8.0**.

## 1. La commande est un adaptateur

Le shell fournit arguments et options ; la commande les traduit pour un service qui porte le traitement.
Le service retourne un résultat structuré : lignes lues, acceptées, rejetées et erreurs localisées.
La commande décide de l'affichage et du statut de sortie ; le service ne reçoit pas obligatoirement `SymfonyStyle`.
Cela permet de tester les règles métier sans comparer des bordures ASCII ni simuler un terminal.
Autowiring et autoconfiguration détectent une classe service munie de `#[AsCommand]`.
Le nom `app:contacts:inspect` doit être stable : un script peut en dépendre aussi fortement qu'un client dépend d'une URL.

```php
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Attribute\Argument;
use Symfony\Component\Console\Attribute\Option;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Style\SymfonyStyle;

#[AsCommand(name: 'app:hello', description: 'Afficher une salutation.')]
final class HelloCommand
{
    public function __invoke(
        SymfonyStyle $io,
        #[Argument(description: 'Destinataire')] string $name,
        #[Option(description: 'Salutation détaillée')] bool $formal = false,
    ): int {
        $io->writeln(($formal ? 'Bonjour ' : 'Salut ').$name);

        return Command::SUCCESS;
    }
}
```

Les commandes invocables et `#[Argument]`/`#[Option]` existent depuis **7.3** : elles sont donc utilisables en 8.0.
Cette classe n'étend pas `Command` : Symfony adapte sa méthode `__invoke()` ; le typage des paramètres décrit ses entrées.
Une commande classique étendant `Command` reste valable, avec `configure()` et `execute(InputInterface, OutputInterface): int`.
Si elle définit un constructeur pour injecter un service, appeler `parent::__construct()`.

## 2. Arguments positionnels et options nommées

`app:hello Ada --formal` contient un argument `name` et une option booléenne.
Un argument requis n'a pas de valeur par défaut ; un argument optionnel en a une ou accepte `null`.
Les arguments positionnels obligatoires précèdent les optionnels ; un argument tableau se place à la fin.
Une option peut être absente : « valeur requise » signifie qu'une option **présente** attend une valeur, pas qu'elle est obligatoire.
Pour les attributs, le nom est dérivé du paramètre en kebab-case ; un `name:` explicite stabilise le contrat.
Un booléen `false` produit un flag simple ; une option chaîne a sa valeur par défaut explicitement déclarée.
Dans l'API classique, `InputOption::VALUE_NONE`, `VALUE_REQUIRED`, `VALUE_OPTIONAL` décrivent des contrats différents.
`VALUE_IS_ARRAY` se combine avec un mode à valeur ; répéter l'option collecte plusieurs valeurs.
`VALUE_NEGATABLE` permet `--foo` et `--no-foo` : définir et tester le sens de l'absence.
Pour un nom de fichier commençant par `-`, le séparateur `--` termine l'analyse des options.

```bash
symfony console app:hello --help
symfony console app:hello Ada --formal
symfony console app:hello -- -nom
```

## 3. Codes de sortie et erreurs

| Constante | Valeur | Contrat conseillé dans les ateliers |
|---|---:|---|
| `Command::SUCCESS` | 0 | traitement conforme, même si aucun élément à traiter |
| `Command::FAILURE` | 1 | erreur opérationnelle ou lignes rejetées selon contrat |
| `Command::INVALID` | 2 | paramètres métier de commande invalides |

Ces constantes sont des conventions utiles ; toutes les exceptions de parsing ne sont pas automatiquement transformées en `INVALID`.
Ne pas afficher « erreur » puis retourner `SUCCESS` : un ordonnanceur ne lit pas ton intention.
Une exception inattendue doit rester diagnostiquable ; la convertir partout en succès masque les régressions.
Un fichier absent, un fichier illisible et un CSV de structure incorrecte méritent des messages distincts.
Le rapport doit préciser la numérotation des lignes : dans le fil rouge, l'en-tête est ligne 1 et la première donnée ligne 2.
Un `--dry-run` n'est pas fourni automatiquement par Symfony : la commande et le service doivent définir ses effets.

## 4. Sortie humaine, sortie machine et verbosité

`SymfonyStyle` fournit titres, tableaux, listes, erreurs et questions avec une présentation cohérente.
Un tableau de synthèse convient à l'humain ; un rapport JSON stable convient à un autre programme.
Éviter barre de progression et messages décorés sur le même flux qu'un JSON consommé par une machine.
Avec `ConsoleOutputInterface`, le flux d'erreur est accessible séparément via `getErrorOutput()`.
`-v`, `-vv`, `-vvv` augmentent la verbosité ; `-q` réduit les sorties, `--silent` supprime aussi les erreurs affichées.
Le mode silencieux ne transforme pas un échec en succès : le statut reste indispensable.
En test, préférer des fragments métier stables aux espaces, couleurs et dimensions du terminal.
Si du texte externe peut contenir des balises de formatage Console, l'échapper avec `OutputFormatter::escape()`.

## 5. Interactions et cycle d'exécution

Pour une commande classique : `initialize()`, puis `interact()` si interactif, puis validation des entrées et `execute()`.
`interact()` peut compléter un argument absent avant la validation finale ; une question n'est pas une validation métier complète.
`--no-interaction` / `-n` empêche ce dialogue : une entrée essentielle doit être fournie ou produire un échec explicite.
Une confirmation doit avoir un défaut prudent et ne pas être la seule façon de lancer une tâche automatisée.
Les ateliers évitent toute mutation durable ; la confirmation sert à apprendre un contrat CLI reproductible.
`CommandTester::setInputs(['yes'])` simule une confirmation positive pour `SymfonyStyle::confirm()` standard ; tester aussi le refus (`no`) et l'absence d'interaction. La question standard reconnaît une réponse commençant par `y`, pas automatiquement `oui` : une confirmation française exige une question configurée pour l'accepter.
Avec les invocables, `#[Interact]` et `#[Ask]` existent depuis 7.4 ; commencer par une seule stratégie lisible.

## 6. Événements Console

L'application dispatch `ConsoleEvents::COMMAND` avant exécution, `ERROR` lors d'une erreur et `TERMINATE` en fin de traitement.
Un listener `ConsoleCommandEvent` peut empêcher l'exécution ; un listener de terminaison peut modifier le code de sortie.
Un listener d'erreur peut observer ou transformer l'erreur ; il ne doit pas masquer un incident involontairement.
Utiliser `#[AsEventListener]` sur un service ; observer les listeners existants avant d'ajouter une configuration.
Les signaux et arrêts brutaux ne garantissent pas un chemin de terminaison ordinaire : ce n'est pas un mécanisme de transaction.
`CommandTester` exécute la commande directement et **ne déclenche pas** les événements de l'application Console.
Pour ceux-ci, utiliser `ApplicationTester` avec l'application FrameworkBundle et désactiver son auto-exit.

## 7. Tester le contrat visible

```php
use Symfony\Bundle\FrameworkBundle\Console\Application;
use Symfony\Component\Console\Tester\CommandTester;

// Dans une méthode d'un KernelTestCase :
self::bootKernel();
$application = new Application(self::$kernel);
$tester = new CommandTester($application->find('app:hello'));
$tester->execute(['name' => 'Ada'], ['interactive' => false]);
$tester->assertCommandIsSuccessful();
self::assertStringContainsString('Salut Ada', $tester->getDisplay());
```

Ce test couvre l'enregistrement réel et la sortie ; les tests du service import couvrent les résultats structurés.
`capture_stderr_separately => true` permet de distinguer les flux via les options de `execute()`.
Chaque test qui crée un CSV ou un rapport possède son répertoire temporaire et le nettoie en `finally`/`tearDown()`.
Ajouter un cas fichier vide, un cas lignes mixtes et un cas option incohérente : un happy path ne définit pas un outil d'automatisation.

## 8. Observer et situer les versions

```bash
symfony console list app
symfony console help app:hello
symfony console debug:container --tag=console.command
symfony console debug:event-dispatcher console.command
symfony console debug:event-dispatcher console.error
symfony console lint:container
php bin/phpunit tests/Console
```

Remplacer `symfony console` par `php bin/console` si nécessaire ; lire `var/log/dev.log` lors d'un échec dans l'application.
**8.0 :** `Application::add()` a été retirée, utiliser `addCommand()` en assemblage manuel ; les anciennes méthodes de nom par défaut ont été retirées.
**8.1 hors certification :** `#[AsCommand]` sur des méthodes, `ConsoleBundle`, `#[AskChoice]`, `CommandTester::run()` et l'API `ExecutionResult`.
Les assertions `assertCommandFailed()` et `assertCommandIsInvalid()` sont aussi nouvelles en 8.1 ; `getStatusCode()` fonctionne pour le socle 8.0.
Lire [Console 8.0](https://symfony.com/doc/8.0/console.html), [entrées](https://symfony.com/doc/8.0/console/input.html) et [événements](https://symfony.com/doc/8.0/components/console/events.html).
Vérifier localement `vendor/symfony/console/CHANGELOG.md`, `Attribute/Argument.php`, `Attribute/Option.php` et `Tester/CommandTester.php`.
Suite : [tests et diagnostic](../14-tests-diagnostic/README.md).
