# Approfondissement — choisir ce que le cache peut conserver

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Une bonne clé et un TTL ne suffisent pas : il faut définir le sens et la durée de vie de la valeur.
Ce complément distingue absence métier, résultat provisoire, panne et objet mutable.
Cache est installé ; les expériences utilisent un `ArrayAdapter` neuf, sans Redis ni Doctrine.
Les contrats présentés existent dans Symfony 8.0 ; une différence de constructeur 8.1 est signalée plus bas.
Les appels de services restent testés depuis un `KernelTestCase`, conformément au parcours.

## 1. Absence et indisponibilité ne décrivent pas la même réalité

Une source peut confirmer qu'un produit n'existe pas ou échouer avant de pouvoir répondre.
Dans le premier cas, `null` peut représenter une réponse métier valide et mémorisable.
Dans le second, mémoriser `null` transformerait une panne en absence apparemment certaine.
Une panne doit donc conserver une représentation distincte : exception ou résultat explicitement indisponible.
Ce choix détermine si le prochain appel interroge à nouveau la source.
Un TTL court pour l'absence confirmée permet de revoir bientôt un produit nouvellement créé.
Le TTL d'une fiche présente peut être plus long si le contrat de fraîcheur le permet.

Le callback des Cache Contracts reçoit un second argument par référence :

```php
use Symfony\Contracts\Cache\ItemInterface;

// Fragment indépendant des services des exercices.
$value = $cache->get('demo.transient', static function (ItemInterface $item, bool &$save): string {
    $save = false;

    return 'Résultat provisoire';
});
```

Le résultat revient à l'appelant, mais ce calcul n'est pas enregistré par cet appel.
Cette possibilité sert lorsqu'une réponse provisoire reste utile sans devoir devenir un hit futur.
Elle ne supprime pas automatiquement une ancienne valeur déjà présente.
Une invalidation et une décision de ne pas enregistrer un calcul sont deux opérations différentes.
Si le callback lève une exception, aucun nouveau résultat de ce callback n'est disponible à mémoriser.
Ne transforme pas toutes les exceptions en valeur vide uniquement pour faire réussir le parcours.
Les [Cache Contracts](https://symfony.com/doc/8.0/components/cache.html) décrivent le calcul à la demande.

## 2. Une valeur mise en cache doit garder son sens

Un objet PHP mutable peut changer après avoir été retourné à un appelant.
Si plusieurs lectures retrouvaient exactement le même objet en mémoire, une modification locale contaminerait les suivantes.
Un stockage sérialisé recrée normalement une valeur distincte à la lecture ; un stockage de références peut se comporter autrement.
Un tableau de scalaires ou un DTO `readonly` exprime plus clairement une fiche calculée qu'un service mutable.
`readonly` ne rend toutefois pas automatiquement immuables les objets mutables contenus dans ses propriétés.
Un DTO `readonly` contenant une `DateTime` mutable peut encore exposer une date modifiable.
Préférer `DateTimeImmutable` et des valeurs simples pour un instant mémorisé.

Un exemple de représentation indépendante du mécanisme de stockage :

```php
$snapshot = [
    'id' => 71,
    'name' => 'Crayon',
    'computedAt' => '2031-04-05T12:00:00+00:00',
];
```

Cette date est une donnée du résultat, pas l'échéance interne de l'entrée de cache.
Le consommateur doit savoir s'il reçoit un instant de calcul, une date métier ou une expiration.
Un objet transport HTTP, un handle de fichier ou un service du conteneur n'est pas une fiche produit.
Le cache doit pouvoir reconstruire sa valeur depuis la source, sans restaurer des connexions ou un contexte de requête.
Une évolution de structure exige aussi de distinguer les anciennes entrées, par exemple par une version dans la clé.

**Différence vérifiée :** le constructeur d'`ArrayAdapter` en **8.0.0** nomme son second argument `storeSerialized`.
Dans le `vendor` **8.1.8** de ce dépôt, cet argument s'appelle `deepClone` et utilise un clonage profond.
Ne copie pas un argument nommé d'une version vers l'autre sans lire la signature installée.
Pour ces ateliers, `new ArrayAdapter()` conserve les valeurs par défaut adaptées et évite cette dépendance de syntaxe.
Le [code officiel 8.0.0](https://github.com/symfony/cache/blob/v8.0.0/Adapter/ArrayAdapter.php) permet la comparaison.
La [documentation de l'adaptateur mémoire 8.0](https://symfony.com/doc/8.0/components/cache/adapters/array_cache_adapter.html) situe ses limites.

## 3. Observer valeur, hit et appel source ensemble

Vérifier seulement le texte retourné peut laisser passer un cache qui ne stocke jamais rien.
Vérifier seulement un compteur peut laisser passer une mauvaise valeur réutilisée pour tous les produits.
Associe résultat exact, nombre d'appels source et scénario temporel ou d'invalidation.
Pour une valeur provisoire, deux lectures identiques doivent pouvoir appeler deux fois la source.
Pour une absence confirmée mémorisée, deux lectures doivent au contraire ne l'appeler qu'une fois.
Lorsqu'un objet mutable revient, modifie la première copie puis vérifie le sens de la lecture suivante.
Une assertion d'identité PHP seule serait trop liée à la représentation choisie.

| Réponse de la source | Sens | Politique à tester |
|---|---|---|
| Fiche complète | Valeur dérivée stable | Hit réutilisable selon TTL |
| Absence confirmée | Résultat négatif réel | `null` mémorisé si c'est le contrat |
| État provisoire | Information utilisable mais instable | Retour possible sans stockage |
| Exception d'indisponibilité | Aucune réponse fiable | Erreur visible, nouvelle tentative ultérieure |
| Objet mutable partagé | Risque de contamination | Copie, instantané ou valeur immuable |

## Observer dans l'application

```bash
php bin/console debug:container cache.app
php bin/console debug:config framework cache
php bin/phpunit tests/AppCache
```

Pour le laboratoire de valeurs, utilise `ArrayAdapter` neuf à chaque cas et aucun délai réel.
Inspecte `ArrayAdapter::get()`, `save()` et `unfreeze()` dans `vendor/symfony/cache/Adapter/ArrayAdapter.php`.
Inspecte le callback et son argument `$save` dans `vendor/symfony/cache-contracts/CacheInterface.php`.
Conserve les compteurs dans la fausse source du test, pas dans une variable globale commune à toute la suite.
Les [exercices 9 et 10](exercices.md) donnent deux contrats précis pour mettre ces décisions à l'épreuve.
