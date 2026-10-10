# Exercices — Module 05

Code dans `src/HttpCache/` (namespace `App\HttpCache`), tests dans `tests/HttpCache/` (namespace `App\Tests\HttpCache`) : `php bin/phpunit tests/HttpCache`. Écris le test d'abord. Pour les tests de contrôleurs, utilise `WebTestCase` (+ `$client->request(...)`).

## 1. Expiration pure
Crée un contrôleur `PriceController` (route `/prix`) qui renvoie du JSON avec un cache **public**, `max-age` 60 s et `s-maxage` 600 s, sans attribut (via l'API de `Response`).
- Tests : en-tête `Cache-Control` contient `public`, `max-age=60`, `s-maxage=600` ; `Response::isCacheable()` vrai.
- Réflexion : qu'est-ce qu'un navigateur retient ? Un CDN ? Que se passe-t-il pour un cache partagé si on enlève `public` mais qu'il y a un cookie de session ?

## 2. `#[Cache]`
Reproduis l'exercice 1 sur une seconde route `/prix-attr` avec l'attribut `#[Cache]` uniquement.
- Test : mêmes en-têtes. Ajoute `vary: ['Accept-Language']` et vérifie `Vary`.
- Réflexion : quel écouteur applique l'attribut ? Retrouve-le avec `debug:event-dispatcher`. Que fait `if:` ?

## 3. Validation par ETag
Crée `ArticleController::show` qui calcule un ETag à partir d'une version (valeur factice dans un service `ArticleRepository` en mémoire). Si `If-None-Match` correspond : **304 sans exécuter le rendu coûteux**.
- Tests : 1re requête 200 + `ETag` ; 2e requête avec `HTTP_IF_NONE_MATCH` ⇒ 304 et contenu vide ; après changement de version ⇒ 200.
- Astuce de test : compte les appels au « rendu coûteux » (double / compteur) pour prouver qu'il n'est pas appelé en 304.
- Réflexion : ETag fort ou faible ici ? Que se passe-t-il si tu appelles `isNotModified()` après le rendu ?

## 4. Last-Modified + `MockClock`
Ajoute `Last-Modified` et gère `If-Modified-Since`. Injecte `ClockInterface` pour contrôler la date en test.
- Tests : date identique ⇒ 304 ; date plus récente côté serveur ⇒ 200.
- Réflexion : ETag et Last-Modified ensemble : lequel prime si les deux sont envoyés par le client ?

## 5. Service `CachePolicy`
Écris une petite classe `CachePolicy` qui applique à une `Response` une politique nommée (`static`, `listing`, `private_user`, `none`) → `Cache-Control` correspondant.
- Tests unitaires par politique (`private_user` : `private`, jamais `public`, `isCacheable()` faux ; `none` : `no-store`).
- Réflexion : différence entre `no-cache` et `no-store` ; pourquoi `max-age=0, must-revalidate` ?

## 6. Listener de sécurité du cache
Crée un `EventSubscriber` sur `kernel.response` qui force `Cache-Control: private, no-store` si la requête porte `Authorization` ou si la réponse pose un `Set-Cookie`.
- Tests : requête avec `Authorization` ⇒ réponse `no-store` ; sans ⇒ inchangé.
- Réflexion : quelle priorité d'écouteur par rapport à `CacheAttributeListener` ? Pourquoi `HttpCache` a déjà `private_headers` ?

## 7. HttpCache en test
Active `framework.http_cache` (config `when@test` ou dédiée), puis vérifie via un test fonctionnel que la 2e requête sur `/prix` est servie du cache (en-tête `X-Symfony-Cache` avec `trace_level: short`) et que le contrôleur n'est exécuté qu'une fois.
- Réflexion : le store est sur disque : comment éviter la pollution entre tests ? Pourquoi une requête avec `Cookie` n'est-elle pas servie du cache ?

## 8. ESI (observation + petit code)
Active `framework.esi` et `fragments`. Crée une page avec `render_esi(controller(...))` pour un bloc `/bloc` (`max-age` 10 s) dans une page `max-age` 300 s.
- Observe : sans proxy ESI, que rend `render_esi` ? Via `HttpCache` (avec ESI activé), quelles durées pour la page finale ?
- Réflexion : pourquoi `/_fragment` est signée ? Quel est le TTL de la page composée si le fragment est `private` ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.


## Critères complémentaires des exercices 1–8

- **1–2** : vérifier les directives individuellement, pas leur ordre textuel ; `if:` est un complément **8.1**.
- **3–4** : ETag différent + date identique → 200 quand la réponse expose les deux validateurs. Le changement de version doit survivre au cycle de test choisi ; ne pas supposer qu'un service en mémoire survit au redémarrage du Kernel. `MockClock` doit être injecté dans ton service de dates ; il ne remplace pas l'horloge interne de tous les objets.
- **5–6** : figer les règles de chaque politique avant le test ; couvrir Authorization et Set-Cookie séparément et vérifier qu'aucun listener ultérieur ne rétablit `public`.
- **7** : monter explicitement `HttpCache` avec un store temporaire unique et faire passer les deux requêtes par ce proxy. Le `KernelBrowser` applicatif seul ne démontre pas un hit. Réponse publique explicite et cookie : comparer au cas sans public au lieu d'affirmer que tout cookie interdit toujours le cache.
- **8 — hors certification** : distinguer TTL de la page source stockée, du fragment stocké et de la réponse assemblée ; travailler sans Age initial pour le premier calcul, puis introduire Age. Ne pas déduire la fraîcheur finale du seul attribut du contrôleur parent.

## 9. Fraîcheur restante (fondation)

Construis une réponse publique `max-age=60`, `s-maxage=600`, avec Date fixe et `Age: 45`.
- Attendu : budget de fraîcheur navigateur 15 s, proxy 555 s ; `getTtl()` donne 555 dans cette expérience. Avec Age 600, la réponse n'est plus fraîche pour le proxy.
- Preuve : assertions sur valeurs et directives, puis explication écrite de la différence entre les deux caches.
- Indice : `s-maxage` ne décrit pas le navigateur ; Age est déjà consommé.

## 10. Validation contradictoire (application)

Sur l'article versionné, fournis simultanément ETag et Last-Modified. Compte les calculs du corps.
- Attendu : ETag identique → 304/corps vide/aucun rendu ; ETag différent + date identique → 200/un rendu ; date seule identique → 304 ; aucun validateur envoyé → 200.
- Ajoute un test prouvant qu'un POST n'est pas transformé en 304 par `isNotModified()`.
- Indice : la présence de plusieurs en-têtes conditionnels n'implique pas un « ou » logique.

## 11. Variantes sans contamination (diagnostic)

Expose un catalogue français et anglais, varie sur `Accept-Language`, avec ETags propres à chaque représentation. Passe réellement par le proxy de l'exercice 7.
- Attendu : fr → miss, en → miss, fr → hit ; chaque corps garde sa langue ; l'ETag français ne valide pas la représentation anglaise. Les compteurs d'origine totalisent un calcul par variante fraîche.
- Indice : vérifier ensemble clé du cache, Vary et version de représentation ; vider uniquement le store dédié au test.

## 12. Deux utilisateurs authentifiés (approfondissement)

Utilise des doubles ou les outils de connexion du client de test pour deux utilisateurs A et B. Une route personnelle affiche leur identifiant ; une route catalogue reste identique pour tous.
- Attendu : route personnelle `private, no-store`, aucune réponse de A servie à B, compteur d'origine incrémenté pour chaque appel ; catalogue public sans session, deuxième requête éligible au hit.
- Variante : un flash ou un `Set-Cookie` apparaît ; prouver que la politique personnelle reste effective après tous les listeners.
- Indice : ne pas envoyer de vrais identifiants externes ; `Vary: Cookie` n'est pas une autorisation.

## Défi final — catalogue public et compte privé

Compose catalogue multilingue, article versionné et page de compte. Livre une matrice route/utilisateur/langue/headers/statut/hit ou miss/appels d'origine. Attendu : fraîcheur et variantes correctes, 304 sans rendu, 200 après changement de version, aucune fuite entre utilisateurs. Un test de contrôleur vérifie les en-têtes ; un autre avec `HttpCache` prouve le stockage réel. ESI reste une extension facultative hors certification, sans être nécessaire à la réussite du défi.
