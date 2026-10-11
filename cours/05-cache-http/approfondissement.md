# Approfondissement — Revalidation, invalidation et réponses négatives

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Prérequis : ETag, fraîcheur et test explicite de `HttpCache` ; prévoir 40 minutes.
Ces mécanismes relèvent du socle **Symfony 8.0** et ont été relus dans **8.1.8**.
Aucun CDN, service externe, composant Lock ou stockage Doctrine n'est nécessaire à ces expériences.

## 1. Le statut vu par le client peut différer de celui de l'origine

Le navigateur et le reverse proxy ont chacun leur copie et leurs validateurs.
Une requête ordinaire du navigateur peut pousser le proxy à revalider sa copie auprès de l'origine.
Si l'origine renvoie 304, le proxy dispose déjà du corps qu'il doit fournir au navigateur.
Il peut alors répondre 200 avec ce corps : il n'y a aucune contradiction.

```text
Client sans condition → proxy avec copie à revalider
                       proxy → origine : If-None-Match
                       proxy ← origine : 304
Client ← proxy : 200 + corps stocké
```

Si le client a lui-même transmis un validateur approprié, une réponse finale 304 devient possible.
Il faut toujours nommer les deux échanges avant d'interpréter un statut.
La trace du contrôleur seul ne décrit pas nécessairement ce qui sort du proxy.
Dans le code installé, `HttpCache::validate()` fusionne les métadonnées puis réutilise l'entrée stockée.

### `no-cache` est une politique de réutilisation

Pour une réponse publique dotée d'un ETag, `no-cache` demande une validation avant réutilisation.
Le corps peut être stocké et resservi après accord de l'origine.
Le nombre de requêtes vers l'origine augmente donc même si le nombre de rendus coûteux ne change pas.
Avec `no-store`, le cache ne doit pas conserver cette nouvelle réponse : le scénario n'a plus ce corps stocké à valider.

```php
$response->setPublic();
$response->setEtag('inventory-v4');
$response->headers->addCacheControlDirective('no-cache');
```

Compter séparément entrée dans le Kernel d'origine, lecture de version et génération du corps.
Un 304 bien placé économise le rendu, pas toutes les opérations nécessaires pour connaître la version.
Pour l'expérience, envoyer des requêtes clientes sans directives de cache afin d'isoler la politique de la réponse.
Le `Cache-Control: no-cache` d'une **requête** possède un chemin distinct, influencé notamment par l'option `allow_reload` du proxy Symfony.

| Réponse d'origine | Peut conserver le corps ? | Deuxième lecture sans condition cliente |
|---|---|---|
| `public, max-age=600` | Oui | Peut éviter complètement l'origine |
| `public, no-cache` + ETag | Oui | Revalide ; corps réutilisable si version inchangée |
| `private` | Pas dans ce cache partagé | Appelle l'origine |
| `no-store` | Non | Appelle l'origine sans copie nouvellement stockée |

Un changement vers `no-store` n'est pas une purge universelle de toutes les anciennes copies déjà distribuées.
La politique doit être correcte dès la première réponse sensible.

## 2. Une écriture n'invalide pas toutes les pages qui en dépendent

Lorsqu'une méthode non sûre traverse `HttpCache`, le proxy transmet l'opération à l'origine.
Après une réponse réussie ou une redirection reconnue, il invalide l'URI ciblée.
Le code installé examine aussi `Location` et `Content-Location` présents sur cette réponse.
Une réponse d'échec comme 409 ne suit pas ce chemin d'invalidation réussie.

Considérons deux pages stockées : `/lesson/cache/items/7` et `/lesson/cache/items`.
Un PUT réussi sur la première peut invalider sa représentation sans invalider la liste.
Le proxy ne connaît pas le lien métier entre le titre d'un article et toutes les pages qui l'affichent.
Une écriture faite hors de ce proxy ne lui fournit même pas cette observation HTTP.

| Événement | Ce que le proxy peut déduire | Ce qu'il ignore |
|---|---|---|
| PUT réussi sur une URI | L'entrée visée doit être invalidée | Les listes et agrégats métier |
| PUT refusé avec 409 | Aucun succès d'écriture annoncé | Un éventuel effet caché d'un mauvais contrôleur |
| Mise à jour depuis une commande | Rien sans mécanisme additionnel | La totalité du changement |
| TTL court de la liste | Une borne de fraîcheur | L'instant précis de modification |

Définir une politique de cohérence : expiration courte, URLs versionnées ou invalidation adaptée au proxy utilisé.
Ne pas envoyer une requête `PURGE` en supposant qu'elle possède une implémentation universelle.
Ne pas placer arbitrairement une URL de liste dans `Location` uniquement pour détourner son sens HTTP.
Les liens retournés doivent continuer à décrire correctement la réponse de l'opération.

La preuve de l'invalidation n'est pas seulement un en-tête de diagnostic.
Après la modification, rappeler réellement le détail via le proxy et vérifier nouvelle version et appel à l'origine.
Puis rappeler la liste pour mesurer explicitement sa politique indépendante.
Conserver le même proxy et le même store pendant tout le scénario ; les recréer masquerait le problème.

## 3. Une absence peut elle aussi être mise en cache

Une réponse 404 n'est pas obligatoirement éphémère.
Avec une politique publique et une fraîcheur explicite, le cache peut conserver « cet article n'existe pas ».
Cela soulage l'origine sur des identifiants inexistants, mais retarde la découverte d'une ressource créée ensuite.
Le statut seul ne suffit donc pas à choisir la durée ; réfléchir à la fréquence de création et à l'impact du retard.

```php
$response = new Response('introuvable', Response::HTTP_NOT_FOUND);
$response->setPublic()->setMaxAge(15);
```

Ce fragment illustre une absence publique, jamais une page qui révèle l'existence d'une donnée privée.
Une erreur 500 n'a pas la même signification qu'une absence 404 : ne pas leur attribuer une politique unique par commodité.
Dans un test, demander deux fois l'identifiant absent puis le créer, et distinguer création à travers ou hors du proxy.
Une simulation d'horloge doit contrôler les calculs réellement employés ; aucun `sleep()` long n'est nécessaire pour ces exercices.

## Observer dans l'application

```bash
bin/console debug:config framework http_cache
php bin/phpunit tests/HttpCache
```

Lire `lookup()`, `validate()`, `invalidate()` dans `vendor/symfony/http-kernel/HttpCache/HttpCache.php`.
Comparer `Response::isCacheable()` dans `vendor/symfony/http-foundation/Response.php` au statut et aux directives choisis.
Créer un store temporaire par test et le nettoyer après le scénario, même en cas d'assertion en échec.
Conserver une trace `[méthode, URI, condition reçue, statut origine, statut client]` pour expliquer les résultats.

## Documentation officielle

- [Validation HTTP — Symfony 8.0](https://symfony.com/doc/8.0/http_cache/validation.html).
- [Invalidation HTTP — Symfony 8.0](https://symfony.com/doc/8.0/http_cache/cache_invalidation.html).
- [RFC 9111 : invalidation](https://www.rfc-editor.org/rfc/rfc9111.html#section-4.4) et [directives de réponse](https://www.rfc-editor.org/rfc/rfc9111.html#section-5.2.2).
