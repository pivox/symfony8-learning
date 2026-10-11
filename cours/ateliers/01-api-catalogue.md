# Atelier 01 — Un catalogue JSON au contrat vérifiable

[Ateliers](README.md) · [HTTP](../04-http-rfc9110/README.md) · [Serializer](../07-serializer/README.md) · [HttpClient](../15-http-client/README.md)

**Niveau :** intermédiaire. **Durée :** 3–4 h. **Prérequis :** modules 03, 04, 05, 07, 11, 12, 14 et 15. Code dans `src/Ateliers/Catalogue/`, tests dans `tests/Ateliers/Catalogue/`.

## Mission et données

Construis une API publique de lecture qui transforme la réponse d’un catalogue externe simulé. Le client HTTP reçoit cette fixture ; toutes les requêtes passent par `MockHttpClient` dans les tests.

| id | title | priceCents | internalCostCents |
|---|---|---|---|
| 41 | Symfony pratique | 3200 | 1100 |
| 42 | HTTP expliqué | 2400 | 700 |
| 43 | Symfony et les tests | 2800 | 900 |

Le fournisseur répond à `GET https://catalogue.example.test/books` par un tableau JSON de ces trois objets. Son coût interne sert à vérifier l’absence de fuite : il ne fait jamais partie de la représentation publique.

L’API locale expose `GET /training/catalogue/books` et `GET /training/catalogue/books/{id}`. Les routes sont en attributs, au format JSON, avec des noms préfixés `atelier_catalogue_`. Les IDs de chemin contiennent uniquement des chiffres. Pas de persistance, session ou authentification.

## Jalon 1 — Définir les frontières

Sépare le client fournisseur, la recherche métier et la représentation publique. Le contrôleur ne décode pas manuellement le corps entrant et ne contient pas la logique de filtrage. Utilise le Serializer pour la sortie et un DTO de lecture ou des groupes explicites pour exposer uniquement `id`, `title`, `priceCents`.

Écris d’abord un test de service qui vérifie méthode/URL du fournisseur, puis résultat exact. Ajoute une réponse 503, un JSON cassé et un produit sans `title`. Définis une exception métier pour l’indisponibilité et une autre pour une réponse fournisseur invalide ; conserve la cause quand elle existe. Aucun de ces défauts ne doit se transformer en liste vide.

**Validation :** un appel fournisseur par opération, aucun réseau réel, trois objets publics distincts, aucun champ interne. La chaîne JSON publique ne doit pas contenir `internalCostCents`.

## Jalon 2 — Recherche et pagination

La collection accepte `q` (chaîne, vide par défaut), `page` (entier, défaut 1, minimum 1) et `limit` (entier, défaut 2, intervalle 1–10). La recherche est une sous-chaîne insensible à la casse sur les titres de cette fixture ; tri final par ID croissant avant pagination.

Mappe la query string vers un DTO validé avec `#[MapQueryString]`. Choisis explicitement **422** pour les violations. Pour une query vide, prévois une valeur par défaut de l’argument ; ne dépends pas d’une option ajoutée après Symfony 8.0.

Le résultat contient `items`, `total`, `page`, `limit`. `total` est le nombre de résultats après filtrage et avant pagination.

| Requête | Attendu |
|---|---|
| Sans query string | IDs `[41, 42]`, total 3, page 1, limit 2 |
| `?q=symfony&limit=1&page=2` | ID `[43]`, total 2 |
| `?q=absent` | Liste vide, total 0, 200 |
| `?page=9` | Liste vide, total 3, 200 |
| `?page=0` ou `?limit=11` | 422 ; aucun appel fournisseur |
| Détail 42 | Objet public du livre 42, 200 |
| Détail 99 | 404 après recherche, aucun produit inventé |
| Détail `abc` | 404 de routage, aucun appel fournisseur |
| POST sur la collection | 405 et `Allow` contenant GET |

Utilise des tests HTTP pour distinguer rejet avant action et absence métier. Les champs d’erreur peuvent être ceux du framework ; ne verrouille pas une stack trace ou une traduction entière dans les assertions.

## Jalon 3 — Réponse conditionnelle

Ajoute un ETag sur le **détail réussi**. Sa valeur dépend de la représentation publique : deux réponses identiques produisent le même tag ; un changement de prix public produit un tag différent. Le corps et le tag sont calculés de manière déterministe.

Pour ce laboratoire public, annonce `public`, `max-age=60` et `s-maxage=120`, puis applique la condition de la requête. Les erreurs ne reçoivent pas cette politique publique. Le module Cache HTTP explique la différence entre un en-tête et un cache effectivement installé.

**Validation :** première requête → 200 et ETag ; seconde avec `If-None-Match` identique → 304, corps vide et ETag conservé ; tag différent → 200. Une fixture dont le prix public change invalide l’ancien tag. Une modification du seul coût interne ne le change pas. HEAD sur le détail retourne un corps vide et les métadonnées attendues.

Ce test prouve le contrat du serveur d’origine, pas l’absence d’appel au fournisseur pendant une revalidation ni le fonctionnement d’un reverse proxy. N’ajoute pas un cache applicatif pour masquer cette distinction.

## Incident à diagnostiquer

Le détail 42 fonctionne, mais la recherche `?q=symfony&page=0` retourne 404 après ajout du DTO. Reproduis d’abord le problème avec cette pagination invalide et le statut par défaut de `MapQueryString`. Identifie qui choisit ce statut, puis rétablis le contrat 422 et ajoute le test qui le protège. La recherche valide `?q=symfony` reste en 200.

Deuxième expérience : expose volontairement l’objet fournisseur entier dans une branche locale de l’exercice. Le test d’absence du coût interne doit échouer. Rétablis ensuite la représentation publique.

## Indices progressifs

<details>
<summary>Indice 1 — Où placer chaque règle ?</summary>

La route valide la forme du chemin ; le Validator vérifie les valeurs de recherche ; le service sait si le livre existe ; le Serializer contrôle les champs publics. Une règle n’a pas besoin d’être répétée dans ces quatre couches.

</details>

<details>
<summary>Indice 2 — Que regarder dans Symfony ?</summary>

Observe `MapQueryString::$validationFailedStatusCode`, le chemin « payload absent » de `RequestPayloadValueResolver`, puis `Response::setEtag()` et `Response::isNotModified()`. Le client de test peut rejouer les en-têtes de la première réponse sans connaître d’avance la valeur du tag.

</details>

## Livraison personnelle

- [ ] Matrice HTTP couverte, statuts et contenus vérifiés.
- [ ] Champ interne absent et erreurs fournisseur explicites.
- [ ] Tests 200/304/HEAD avec au moins une variation de représentation.
- [ ] Explication de ce qui peut encore être recalculé lors d’un 304.

Références : [mapping des arguments](https://symfony.com/doc/8.0/controller.html#mapping-the-whole-query-string), [validation HTTP conditionnelle](https://symfony.com/doc/8.0/http_cache/validation.html), [tests HttpClient](https://symfony.com/doc/8.0/http_client.html#testing). Les tests s’appuient sur les classes installées `RequestPayloadValueResolver`, `Response` et `MockHttpClient`.
