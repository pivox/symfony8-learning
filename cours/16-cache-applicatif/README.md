# Module 16 — Cache applicatif

[Parcours](../README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Réduire les calculs répétés, définir une clé correcte et invalider les données au bon moment. Prérequis : [DI](../03-dependency-injection/README.md), [Clock](../10-composants/README.md), [tests](../14-tests-diagnostic/README.md).

`symfony/cache` est déjà installé comme dépendance. Aucun serveur Redis n'est nécessaire : les exercices utilisent un pool local ou `ArrayAdapter`.

**Versions :** application Symfony 8.1.8, socle étudié compatible Symfony 8.0. Le [module 05](../05-cache-http/README.md) traite des réponses HTTP ; ici, on mémorise des valeurs utilisées par les services.

## 1. Trois interfaces à distinguer

| API | Modèle d'utilisation | Point de vigilance |
|---|---|---|
| PSR-6 `CacheItemPoolInterface` | Récupérer un item, vérifier le hit, définir et sauvegarder | Une valeur `null` n'est pas forcément un miss |
| PSR-16 `CacheInterface` | `get`, `set`, `delete` de valeurs | Même nom court que Symfony Contracts, namespace différent |
| Symfony Contracts `CacheInterface` | `get` avec callback de calcul | La valeur retournée par le callback est enregistrée |

Le contrat Symfony se trouve dans `Symfony\Contracts\Cache`. Le contrat PSR-16 dans `Psr\SimpleCache`. `psr/simple-cache` n'est pas installé dans le dépôt initial ; l'observation comparative ne nécessite pas son installation. Pour pratiquer son adaptateur : `composer require psr/simple-cache`.

## 2. Calculer à la demande

Exemple minimal indépendant des exercices :

```php
use Symfony\Component\Cache\Adapter\ArrayAdapter;
use Symfony\Contracts\Cache\ItemInterface;

$pool = new ArrayAdapter();
$value = $pool->get('demo.answer', static function (ItemInterface $item): int {
    $item->expiresAfter(60);

    return 42;
});
```

Le callback est appelé en cas d'absence ou de recalcul anticipé selon l'adaptateur. Il ne doit pas porter un effet métier qui doit arriver exactement une fois, comme débiter un compte. Le cache peut être vidé ou indisponible ; le résultat doit pouvoir être reconstruit.

## 3. Clés et isolation

La clé décrit **toutes** les entrées influençant le résultat : produit, langue, devise, client si nécessaire, version de l'algorithme. Une clé seulement basée sur le produit peut servir un tarif ou un libellé d'un autre contexte.

Choisis une représentation stable. Pour des structures complexes, normalise les données avant de les condenser ; le même sens avec un ordre de clés différent ne devrait pas créer arbitrairement deux entrées. Respecte les caractères autorisés par l'API de cache.

Un pool nommé isole ses entrées des autres pools. Pour l'application, configure un pool dans `framework.cache.pools` puis injecte-le avec `#[Autowire(service: 'catalogue.cache')]`. Cette configuration complète FrameworkBundle déjà installé ; elle ne remplace pas une recette de bundle.

## 4. Expiration et horloge

Une durée d'expiration limite la fraîcheur ; elle ne garantit pas que le calcul ne sera pas refait avant. `expiresAfter(0)` expire immédiatement, tandis qu'une durée par défaut de zéro dans certains adaptateurs signifie absence de limite : ne mélange pas les deux paramètres.

Pour tester une échéance sans attendre, `ArrayAdapter` accepte une horloge PSR-20. Utilise `ArrayAdapter(defaultLifetime: 60, clock: $clock)` et ne fixe pas d'expiration explicite dans le callback de ce test. L'adaptateur calcule alors l'échéance avec cette horloge.

Attention : dans la version installée, `CacheItem::expiresAfter()` calcule une date à partir de `microtime(true)` ; lui associer seulement un adaptateur doté de `MockClock` mélange deux horloges. Pour une expiration propre à l'item, utilise `expiresAt()` avec une date issue de la même `MockClock`. L'exemple précédent avec `expiresAfter(60)` illustre l'usage normal en temps réel, pas ce laboratoire temporel.

## 5. Invalidation ciblée et tags

`delete()` invalide une clé. `TagAwareCacheInterface::invalidateTags()` invalide toutes les entrées associées aux tags demandés : c'est utile pour des pages de listes dépendant d'un même produit.

Pour utiliser `$item->tag(...)`, il faut un pool compatible avec les tags, par exemple `TagAwareAdapter`. Un pool ordinaire ne devient pas compatible parce que le callback appelle `tag()`.

L'invalidation est une décision métier : après une modification réussie des données de référence, les anciennes valeurs dérivées doivent disparaître. Un test prouve aussi que les autres groupes restent en cache.

## 6. Recalcul anticipé et concurrence

Les adaptateurs concernés peuvent anticiper l'expiration pour éviter que de nombreuses requêtes recalculent simultanément. Dans le contrat, `beta: 0` désactive l'expiration anticipée probabiliste et `beta: INF` force un recalcul. Lis l'adaptateur choisi : `ArrayAdapter` ne reproduit pas la coordination entre processus.

Ne conclus pas « protégé contre toutes les courses distribuées » après un test en mémoire. La protection dépend du stockage, des processus et des mécanismes de coordination utilisés. Les verrous métier feront l'objet d'un module professionnel distinct.

## Observer dans l'application

```bash
php bin/console cache:pool:list
php bin/console debug:config framework cache
php bin/console debug:container cache.app
php bin/console debug:autowiring 'Symfony\Contracts\Cache'
php bin/phpunit tests/AppCache
```

Les commandes `cache:pool:clear` et `cache:pool:invalidate-tags` modifient les données : les pratiquer uniquement sur le pool dédié aux exercices. Ne vide pas tous les pools pour masquer un défaut d'invalidation.

## Sources

- [Cache Symfony](https://symfony.com/doc/current/cache.html), [composant Cache](https://symfony.com/doc/current/components/cache.html).
- [PSR-6](https://www.php-fig.org/psr/psr-6/) et [PSR-16](https://www.php-fig.org/psr/psr-16/).
- Sources installées : `vendor/symfony/cache-contracts/CacheInterface.php`, `vendor/symfony/cache/Adapter/ArrayAdapter.php`, `vendor/symfony/cache/Adapter/TagAwareAdapter.php`.
