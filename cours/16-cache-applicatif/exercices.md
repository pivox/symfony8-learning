# Exercices — Module 16

[Cours](README.md) · [Quiz](quiz.md) · [Méthode](../README.md)

Code dans `src/AppCache/`, tests dans `tests/AppCache/` avec `KernelTestCase` pour les services. Chaque test reçoit un pool neuf. Fil rouge : calcul de fiches produit depuis une source en mémoire avec compteur d'appels.

## 1. Guidé — prouver l'économie de calcul

Crée un service de fiche produit qui mémorise le résultat de la source avec les Cache Contracts.

- **Critères :** deux demandes identiques retournent les mêmes données ; la source est appelée une fois. Un autre produit provoque un deuxième calcul.
- **Indice :** un compteur dans une fausse source prouve davantage qu'un simple résultat correct.

## 2. Guidé — une clé qui représente le contexte

Ajoute une langue `fr` ou `en`. Pour le produit 42, la source retourne `Carnet` ou `Notebook`.

- **Critères :** appels successifs `(42, fr)`, `(42, en)`, `(42, fr)` donnent les trois résultats attendus et exactement deux calculs. Une langue non supportée est refusée avant accès à la source.
- **Indice :** écrire le test de contamination entre langues avant de changer la clé.

## 3. Autonome — échéance déterministe

Fixe une durée de vie de 60 secondes avec `ArrayAdapter(defaultLifetime: 60, clock: $clock)` et une `MockClock` figée. Le callback de ce test ne définit pas d'expiration explicite. Variante : calcule une date `expiresAt()` avec cette même horloge.

- **Critères :** à t0 et t+59 secondes, un seul calcul ; à t+61, un second. Le test ne dort pas réellement et contrôle l'horloge du pool.
- **Indice :** compare l'argument `clock` du constructeur avec `CacheItem::expiresAfter()` : celui-ci lit encore le temps réel dans la version installée.

## 4. Diagnostic — absence et valeur nulle

Représente un produit absent par `null`. Compare une implémentation PSR-6 correcte à une variante qui teste uniquement `get() === null`.

- **Critères :** deux lectures d'un absent ne sollicitent la source qu'une fois dans la version correcte ; ton test fait échouer la variante. Utilise `isHit()` pour observer la différence.
- **Indice :** une entrée existante peut contenir `null`.

## 5. Autonome — invalider un groupe

Mémorise les fiches 42 et 43 dans les deux langues ; associe les deux fiches 42 au tag `product.42`, les deux autres à `product.43`.

- **Critères :** après invalidation de `product.42`, seules ses deux variantes sont recalculées ; 43 reste servi sans nouvel appel. Une invalidation d'un tag inconnu ne casse pas les lectures.
- **Indice :** utilise un adaptateur compatible avec les tags ; vérifie les appels par produit, pas seulement le total.

## 6. Autonome — isoler deux pools

Déclare `catalogue.cache` et `report.cache`. Enregistre des valeurs différentes sous une même clé logique dans les deux pools.

- **Critères :** valeurs distinctes, injection explicite dans deux services, suppression d'une entrée dans le premier sans effet sur le second. Utilise des pools de test sans données personnelles.
- **Indice :** un identifiant de service n'est pas une clé d'entrée.

## 7. Diagnostic — forcer un recalcul

Avec un pool `ArrayAdapter`, appelle deux fois une clé avec `beta: 0`, puis une troisième fois avec `beta: INF`.

- **Critères :** compteur égal à 1 avant l'appel forcé, puis 2. Dans un cas séparé sur une clé absente, une exception du callback remonte ; l'appel suivant retente le calcul. Explique pourquoi ce test ne prouve pas une coordination entre workers.
- **Indice :** le paramètre contrôle le recalcul, pas le délai réseau de la source.

## 8. Défi final — catalogue mis à jour

Combine source en mémoire, langue, TTL et tags. Lis deux produits dans deux langues, modifie le nom du produit 42 dans la source, invalide son tag, puis relis les quatre fiches.

- **Critères :** 42 reflète la modification ; 43 ne se recalcule pas ; après l'échéance les fiches demandées se recalculent. Tous les compteurs sont vérifiés, sans attente réelle.
- **Livrable :** tests plus un tableau « entrée du calcul → partie de la clé → événement d'invalidation ».
- **Indice :** la source reste l'autorité ; le cache contient uniquement une valeur dérivée.
