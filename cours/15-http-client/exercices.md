# Exercices — Module 15

[Cours](README.md) · [Quiz](quiz.md) · [Méthode](../README.md)

Code à écrire dans `src/HttpClient/`, tests dans `tests/HttpClient/`. Tests de services : `KernelTestCase`. Toutes les réponses proviennent de `MockHttpClient`, sans accès réseau. Écris d'abord les assertions ; les noms des exceptions métier ci-dessous sont des classes à créer.

## 1. Guidé — premier contrat externe

Crée `CatalogueClient::find(int $id): ?array`. Pour l'identifiant 42, il appelle `GET https://catalogue.example/v1/products/42`. Un 200 avec `{"id":42,"name":"Carnet"}` retourne ces deux champs ; un 404 retourne `null`.

- **Critères :** méthode et URL vérifiées par la factory du mock ; résultat exact ; un appel par recherche.
- **Indice :** commencer par `getStatusCode()` évite de traiter un 404 comme une panne réseau.

## 2. Guidé — paramètres et corps JSON

Ajoute `search(string $term): array`, appelant `/v1/products` avec le paramètre `q`, et `reserve(int $id, int $quantity): void`, envoyant un POST JSON à `/v1/reservations` et attendant 201.

- **Critères :** le terme `stylo bleu & noir` arrive intact après décodage des paramètres ; le corps du POST contient les deux entiers ; quantité nulle ou négative refusée avant tout appel.
- **Indice :** inspecter les options normalisées du mock, pas une chaîne construite à la main.

## 3. Autonome — classifier les erreurs

Pour `find()`, définis `CatalogueUnavailable` pour les pannes réseau et statuts 5xx ; `UnexpectedCatalogueResponse` pour un 200 au JSON illisible, sans `id`, ou avec un `id` chaîne. Les autres statuts inattendus produisent aussi `UnexpectedCatalogueResponse`.

- **Critères :** un test distinct pour 503, panne de transport, JSON cassé, champ absent et mauvais type ; 404 reste `null` ; conserve l'exception d'origine lorsqu'il y en a une.
- **Indice :** le décodage réussi ne valide pas le schéma de la réponse.

## 4. Autonome — client dédié

Configure `catalogue.client` avec une base `/v1/`, un timeout de 2 secondes et une durée maximale de 5 secondes. Injecte ce client dans le service, et conserve les chemins de requête relatifs.

- **Critères :** service résolu par le container ; options observables ; aucun réseau dans les tests. Une variante utilisant `/products/42` doit révéler la perte de `/v1/`.
- **Indice :** distingue le test du comportement du service et le test du client sélectionné par le container.

## 5. Diagnostic — reprise bornée

Sur la lecture GET seulement, autorise deux reprises après 503, sans délai dans les tests. Simule 503, 503, 200 ; puis trois 503.

- **Critères :** trois appels et succès dans le premier cas ; trois appels et `CatalogueUnavailable` dans le second. Un 404 n'est pas rejoué. `reserve()` reste sans reprise automatique.
- **Indice :** « deux reprises » signifie au maximum trois tentatives, appel initial compris.

## 6. Autonome — résultats multiples

Crée `findMany(array $ids): array`, en lançant toutes les requêtes avant lecture. Pour `[42, 43]`, renvoie une table indexée par ID ; le produit 43 absent correspond à `null`.

- **Critères :** deux requêtes, association correcte des résultats et absence de mélange des réponses. Documente la stratégie choisie si un fournisseur renvoie 503 : pour cet exercice, l'opération entière échoue.
- **Indice :** conserver l'identifiant à côté de chaque objet réponse. Le test n'a pas à prouver une vitesse réseau.

## 7. Diagnostic — réponse interrompue

Avec `MockResponse`, simule un corps reçu en morceaux, puis une erreur de transport pendant la lecture. Ajoute un cas de corps JSON complet reçu en deux morceaux.

- **Critères :** le corps complet est assemblé ; la réponse interrompue ne retourne jamais un produit partiel ; l'erreur est traduite selon l'exercice 3.
- **Indice :** un générateur utilisé comme corps permet de différer les morceaux et l'erreur jusqu'à la consommation. Lis les exemples dans le composant installé.

## 8. Défi final — rapport de disponibilité

Crée un service qui produit un rapport pour `[42, 43, 44]` : 42 existe, 43 est absent, 44 répond 503 puis 200. Réutilise le client et sa politique de reprise.

- **Critères :** rapport de trois lignes, ordre des identifiants conservé, état explicite pour l'absent, quatre requêtes au total et aucun réseau. Un JSON invalide sur 44 fait échouer le rapport avec une erreur explicite.
- **Livrable :** tests et une courte explication de l'endroit où tu traduis les exceptions techniques.
- **Indice :** sépare l'interprétation de l'API de la construction du rapport.
