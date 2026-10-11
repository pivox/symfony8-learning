# Approfondissement — parcours différés et flux de processus

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Le cours présente Finder et Process séparément ; ici, leur point commun est la frontière entre description et exécution.
Configurer une recherche ne fige pas les fichiers, et recevoir des octets ne livre pas forcément une ligne complète.
Socle **Symfony 8.0**, vérifié dans les composants **8.1.8** installés.
Prérequis : itérateurs PHP, exceptions, dossier temporaire et distinction stdout/stderr.
Aucun outil externe n'est exigé : les processus pédagogiques utilisent `PHP_BINARY`.

## 1. Finder décrit une recherche, pas un instantané

`Finder` conserve des critères et produit un itérateur lorsqu'on le parcourt.
Ses méthodes de configuration modifient la même instance et retournent généralement `$this`.
Une affectation `$second = $first` ne crée donc pas une recherche indépendante.
Ajouter ensuite `name()` ou `depth()` peut modifier les observations des deux variables.
Construire un Finder neuf par recherche rend le contrat d'un service plus facile à contrôler.

```php
use Symfony\Component\Finder\Finder;

$finder = Finder::create()->files()->in($directory)->name('*.txt');
// Ici, la recherche est configurée ; son résultat n'a pas été mémorisé.
```

`in()` vérifie déjà que la racine existe : « différé » ne veut pas dire « aucune opération avant foreach ».
Un fichier créé après la configuration mais avant l'itération peut apparaître dans le résultat.
Une seconde itération reconstruit le parcours et peut observer de nouvelles entrées.
`count()` et `hasResults()` déclenchent aussi un parcours ; ils ne constituent pas une réservation des résultats futurs.
Faire `count($finder)` puis `foreach` n'assure pas que le nombre traité sera encore le nombre annoncé.
Le système de fichiers peut évoluer entre ces deux opérations.

### Matérialiser selon le besoin

Pour un rapport déterministe, relever une fois les chemins relatifs et les trier selon une règle définie.
Conserver des `SplFileInfo` ne fige pas forcément le contenu de leurs fichiers : un appel ultérieur à `getContents()` lit encore le disque.
Si le contrat concerne les contenus, copier les contenus au moment choisi et documenter cette limite.
Même cette lecture n'est pas un instantané transactionnel de plusieurs fichiers modifiés concurremment.
Les fixtures de cours restent dans une arborescence temporaire contrôlée pour isoler ce mécanisme.

| Besoin | Représentation à conserver | Limite |
|---|---|---|
| refaire une recherche actuelle | Finder ou nouvelle instance | résultats susceptibles de changer |
| mémoriser une liste de noms | tableau de chemins relatifs | fichiers susceptibles de disparaître |
| mémoriser le texte lu | tableau chemin → contenu | lecture de chaque fichier à un instant différent |
| traiter des milliers d'entrées | itération au fil du parcours | tri global et mémoire à considérer |

`sortByName()` rend l'ordre observable, mais un tri global doit rassembler des résultats avant de les ordonner.
Le coût d'une recherche ne se résume donc pas au nombre de méthodes chaînées.
Un filtre par nom ne lit pas les fichiers comme un filtre `contains()` recherchant dans leur contenu.
Définir d'abord la sélection nécessaire avant d'ajouter des filtres coûteux.
Un rapport vide après suppression d'un fichier est différent d'une erreur de lecture en cours de traitement.
Le service appelant doit choisir un résultat partiel documenté ou un échec complet, pas avaler toute exception.

## 2. Process transporte des octets, pas des messages déjà découpés

Un callback de `run()` reçoit un type de flux et un fragment de sortie.
Un fragment peut contenir une demi-ligne, plusieurs lignes ou une partie d'un caractère UTF-8.
Le découpage dépend des écritures, des buffers et de l'ordonnancement ; le test ne doit pas fixer le nombre de callbacks.
Concaténer les fragments de stdout reconstitue son contenu, idem séparément pour stderr.
Un ordre total entre les fragments des deux flux n'est pas un contrat métier fiable.

```php
use Symfony\Component\Process\Process;

$process = new Process([PHP_BINARY, '-r',
    'fwrite(STDOUT, strtoupper(stream_get_contents(STDIN)));',
]);
$process->setInput("bonjour\n");
$process->mustRun();
// Le résultat est accessible dans getOutput(), sans passage par un shell.
```

`setInput()` convient à une entrée déjà disponible sous forme de chaîne ou de ressource lisible.
Pour une alimentation progressive, `InputStream` permet d'écrire puis de fermer l'entrée.
Ne pas fermer un flux alors que l'enfant attend la fin de fichier peut le laisser bloqué jusqu'au timeout.
Le timeout protège cette attente ; il ne transforme pas une réponse incomplète en succès.
Avec une ressource, maîtriser sa position et sa fermeture dans le test.

### Définir un protocole avant de parser

Si l'enfant renvoie une ligne JSON par résultat, conserver un buffer des octets non encore terminés par `\n`.
Décoder seulement les lignes complètes, puis décider quoi faire d'un reliquat à la fin.
Le contrat peut exiger un dernier saut de ligne et refuser toute ligne tronquée.
Un code de sortie non nul reste un échec même si quelques lignes valides ont été reçues avant.
Dans ce cas, distinguer résultats provisoires et résultats publiés par le service.
Une sortie stderr informative ne doit pas être injectée dans le parseur JSON de stdout.

| Signal observé | Interprétation | Assertion utile |
|---|---|---|
| deux fragments pour une ligne | découpage normal | un seul résultat final |
| deux lignes dans un fragment | regroupement normal | deux résultats dans l'ordre stdout |
| stderr + code 0 | diagnostic possible | succès selon le contrat annoncé |
| ligne JSON tronquée + code 0 | protocole invalide | erreur de protocole explicite |
| lignes valides + code 7 | exécution échouée | aucune publication finale |

`getIncrementalOutput()` lit seulement ce qui n'a pas déjà été retourné par cette méthode.
`getOutput()` expose la sortie conservée complète ; ce sont deux modes de lecture différents.
Ne pas mélanger leurs résultats dans un même accumulateur sous peine de compter deux fois des données.
Ne pas désactiver la sortie si le service dépend de son contenu ou d'un idle timeout.

## Observer dans l'application

```bash
symfony console debug:autowiring Filesystem
symfony console debug:autowiring Clock
php bin/phpunit tests/Components
```

Le Finder et le Process du scénario sont créés avec leurs paramètres propres ; ce ne sont pas des services partagés présupposés.
Journaliser dans le test les chemins relatifs et les octets reconstitués, sans chronométrage exact.
Pour un test de service, étendre `KernelTestCase` ; ajouter si besoin des expériences ciblées sur les composants autonomes.
Créer et supprimer les fixtures dans un dossier temporaire, y compris après une assertion en échec.
Un enfant PHP de quelques lignes suffit : aucun appel réseau, Git, shell interactif ou attente de plusieurs secondes.

Sources : [Finder 8.0](https://symfony.com/doc/8.0/components/finder.html), [Process 8.0](https://symfony.com/doc/8.0/components/process.html), [Filesystem 8.0](https://symfony.com/doc/8.0/components/filesystem.html).
Lecture locale : `vendor/symfony/finder/Finder.php` (`in`, `getIterator`, `count`) et `Iterator/SortableIterator.php`.
Lecture locale : `vendor/symfony/process/Process.php` (`setInput`, `run`, `getIncrementalOutput`) et `InputStream.php`.
`Finder::useUnixPaths()` est **8.1 uniquement** ; les exercices ci-après ne l'exigent pas.
