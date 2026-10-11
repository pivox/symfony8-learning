# Exercices — Module 16

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md) · [Méthode](../README.md)

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

## 9. Autonome — ne pas conserver une réponse provisoire

**Niveau :** intermédiaire · **Durée indicative :** 45 minutes.
**Prérequis :** exercices 1 et 4 ; [valeurs mémorisables](approfondissement.md).

Scénario : la source expose un état `pending` pendant la préparation d'une fiche, puis un état `ready` stable.
Pour le produit 42, configure la fausse source pour retourner successivement `{"state":"pending"}` puis `{"state":"ready","name":"Carnet"}`.
Le service peut retourner `pending` au premier appel, mais seul `ready` doit devenir une entrée réutilisable.
Utilise les Cache Contracts et un pool `ArrayAdapter` neuf, avec `beta: 0` dans ce laboratoire.

- **Critères :** trois lectures réelles donnent `pending`, `ready`, `ready` ; la source est appelée exactement deux fois ; après le premier appel, la clé n'est pas un hit.
- **Critères :** dans un cas distinct, une absence confirmée `null` peut être mémorisée : deux lectures, un seul appel source, en conservant le contrat de l'exercice 4.
- **Cas négatif :** une exception de source au premier appel n'est pas transformée en `null` mémorisé ; après rétablissement, le second appel obtient la fiche et incrémente le compteur.
- **Livrable :** tests depuis `KernelTestCase` et tableau des trois sens `pending`, absence confirmée, panne.
- **Indice 1 :** « retourner une valeur » et « la conserver » ne sont pas une seule décision.
- **Indice 2 :** le callback peut recevoir `bool &$save` ; observe ensuite le hit de la clé et le prochain appel source pour prouver l'effet de ce choix.

## 10. Diagnostic — une modification locale doit rester locale

**Niveau :** intermédiaire · **Durée indicative :** 45 minutes.
**Prérequis :** exercice 1 ; objets PHP et [instantanés de cache](approfondissement.md).

Scénario : le service retourne une fiche mutable comportant `id=42` et `name=Carnet` ; un consommateur modifie le nom pour son propre affichage.
Utilise un `stdClass` comme fixture de source et `new ArrayAdapter()` avec ses valeurs par défaut.
Après le premier appel au service, affecte `Annotation locale` au nom de la copie reçue, puis relis la même fiche.
Ajoute un contrôle pédagogique isolé avec une simple table PHP stockant directement la référence au même objet.

- **Critères :** avec le cache par défaut, la deuxième lecture retourne encore `Carnet`, avec un seul appel source ; la modification du consommateur n'altère pas la fiche mémorisée.
- **Cas négatif :** dans la table conservant une référence, le contrôle reproduit `Annotation locale` lors de la seconde lecture ; le test explique pourquoi ce comportement serait un défaut dans le service.
- **Critères :** compare les valeurs utiles, pas une hypothèse d'identité d'objet ; un test indépendant retrouve la fixture initiale.
- **Livrable :** note proposant ensuite un tableau de scalaires ou un DTO immuable pour représenter cette fiche, sans remplacer tout le stockage de l'application.
- **Indice 1 :** recherche où une copie est réalisée et où une référence mutable reste partagée.
- **Indice 2 :** lis `ArrayAdapter::save()` et la lecture de l'entrée dans ta version ; le nom du second argument diffère entre 8.0.0 et le vendor 8.1.8, donc conserve ici le constructeur sans cet argument nommé.
