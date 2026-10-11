# Approfondissement — PHP : évaluation, copie et journalisation

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Prérequis : interfaces, closures et exceptions du cours ; prévoir 35 minutes de lecture et d'expérimentation.
Les exemples ciblent PHP **8.4** et restent utilisables avec Symfony **8.0** ; aucune API propre à 8.1 n'est nécessaire.
Le manuel PHP est évolutif : une section marquée PHP 8.5 ne devient pas disponible dans notre runtime 8.4.

## 1. Un générateur représente un calcul suspendu

Une fonction contenant `yield` renvoie un `Generator`, pas son tableau de résultats.
Son corps commence lorsque le consommateur demande la première valeur.
Chaque `yield` suspend l'exécution ; la demande suivante reprend au même endroit.
Une exception située après le deuxième `yield` n'apparaît donc pas à la création du générateur.

```php
function letters(): \Generator
{
    yield 'a';
    yield 'b';
}

$sequence = letters(); // Aucun élément n'a encore été consommé.
$first = $sequence->current(); // Lance le calcul jusqu'au premier yield.
```

`iterable` accepte un tableau ou un objet `Traversable` ; ce type ne promet ni indexation ni second parcours.
Un même générateur terminé ne se rembobine pas pour une nouvelle boucle.
Une nouvelle invocation de `letters()` construit en revanche un nouveau parcours indépendant.
Pour un contrat réitérable, fournir une fabrique ou un `IteratorAggregate` créant un générateur à chaque appel.

La paresse évite de produire les éléments inutilisés ; elle ne garantit pas à elle seule une mémoire constante.
Si le producteur charge tout avant son premier `yield`, le coût initial existe toujours.
`iterator_to_array()` matérialise également tous les résultats : pertinent pour un petit test, différent d'un traitement en flux.
Attention à ses clés : deux éléments avec la même clé s'écrasent si leur conservation est activée.

| Besoin du consommateur | Contrat adapté | Point à vérifier |
|---|---|---|
| Un seul passage, arrêt précoce | `iterable` paresseux | Aucun élément produit après l'arrêt |
| Deux passages indépendants | Fabrique ou `IteratorAggregate` | Nouveau parcours à chaque demande |
| Accès aléatoire et comptage immédiat | Tableau matérialisé | Taille acceptable en mémoire |
| Ressource ouverte dans le producteur | `try/finally` | Moment réel de destruction du générateur |

Un `break` ne détruit pas un générateur encore référencé dans une variable.
Son bloc `finally` peut donc attendre sa fermeture effective ; ne pas lui attribuer la durée de vie de la boucle.
Pour observer, journaliser « début », « élément », « fin » dans un tableau de test, sans `echo` métier.
Comparer la trace avant `current()`, après `next()` et après consommation complète.

## 2. Copier une référence, cloner un objet, figer une valeur

`$b = $a` pour un objet fait pointer deux variables sur la même instance.
`clone $a` crée une nouvelle instance externe, mais réalise une copie superficielle de ses propriétés.
Les objets contenus restent donc partagés si le contrat de clonage ne prévoit rien de plus.

```php
$original = (object) ['date' => new \DateTime('2026-01-10 UTC')];
$copy = clone $original;
$copy->date->modify('+1 day');
// Les deux conteneurs observent maintenant le 11 janvier.
```

Le problème n'est pas une mauvaise copie de la date : aucune copie de cet objet interne n'a été demandée.
`__clone()` permet de choisir les propriétés à cloner, après la copie externe.
Une copie profonde automatique de tout le graphe serait parfois incorrecte : un service partagé doit généralement rester partagé.
Le choix appartient au domaine, par exemple conserver un catalogue commun mais isoler un brouillon de panier.

Une propriété `readonly` interdit sa réaffectation ordinaire ; elle n'immobilise pas un objet mutable qu'elle référence.
Pour une valeur datée, `DateTimeImmutable` rend explicite le remplacement par une nouvelle valeur.
Une méthode `with…()` peut retourner un nouvel objet construit avec une seule différence, sans mutation de l'original.
Ne pas utiliser ici la syntaxe `clone($objet, [...])`, introduite en PHP 8.5.

| Vérification | Ce qu'elle démontre | Ce qu'elle ne démontre pas |
|---|---|---|
| `$a === $b` | Identité d'instance | Égalité métier de deux instances distinctes |
| Même contenu après création | Valeurs initiales égales | Absence de partage ultérieur |
| Mutation de la copie puis lecture de l'original | Isolation sur ce chemin précis | Copie profonde de tout le graphe |
| Propriété `readonly` | Affectation contrôlée | Immutabilité des objets contenus |

Dans les tests, modifier effectivement la copie puis relire l'original.
Pour comparer des montants, statuts ou identifiants métier, définir les valeurs pertinentes plutôt que dépendre de `spl_object_id()`.

## 3. PSR-3 : le message et son contexte ont des rôles distincts

Un message stable décrit l'événement ; le contexte porte ses données variables.
Cela permet à une implémentation de les indexer sans analyser une phrase déjà concaténée.

```php
$logger->warning('Import interrompu pour {batch}', [
    'batch' => 'B-24',
    'exception' => $failure,
]);
```

`$failure` représente ici un `Throwable` existant ; la clé conventionnelle `exception` conserve cet objet.
L'interpolation des placeholders dépend de l'implémentation : le contrat PSR-3 ne promet pas un format de fichier particulier.
Un test du service vérifie niveau, message et contexte transmis au logger, sans dépendre d'un rendu Monolog.
Ce dépôt possède `psr/log` ; Monolog reste facultatif et n'est pas présupposé.
Ne transmettre que les données utiles : un identifiant de lot suffit, sans copier credentials ou payload complet.

## Observer dans l'application

```bash
php -v
bin/console debug:autowiring Logger
php bin/phpunit tests/Php84
```

Lire `vendor/psr/log/src/LoggerInterface.php` et `AbstractLogger.php` pour les signatures installées.
Un double de logger peut étendre `AbstractLogger` et enregistrer les appels à `log()`.
Pour les générateurs et les copies, exécuter les exemples depuis un fichier temporaire PHP 8.4 avec autoload si nécessaire.
Les exercices 12–13 demandent des appels de service et des résultats observables, sans base de données.

## Documentation officielle

- [PHP 8.4 : nouveautés et périmètre](https://www.php.net/releases/8.4/fr.php).
- [Manuel PHP : générateurs](https://www.php.net/manual/fr/language.generators.overview.php) et [clonage](https://www.php.net/manual/fr/language.oop5.cloning.php), en respectant les mentions de version.
- [PSR-3 : interface de journalisation](https://www.php-fig.org/psr/psr-3/) et [précision sur `Throwable`](https://www.php-fig.org/psr/psr-3/meta/).
