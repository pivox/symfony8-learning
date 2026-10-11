# Exercices — Module 05

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

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

## 13. Revalidation sans nouveau rendu

**Niveau : avancé · Durée : 50–65 min · Prérequis : exercices 3 et 7, [statuts entre origine et proxy](approfondissement.md).**

Une route `/lesson/cache/stock` renvoie exactement `stock=12`, avec ETag `stock-v1` et politique `public, no-cache`. Construire un scénario passant réellement par une même instance de `HttpCache`, un store temporaire et une origine dont les appels et rendus sont comptés séparément.

Les deux requêtes clientes sont des GET ordinaires sans validateur ni directive Cache-Control. L'origine traite ses conditions avant le rendu ; sa version reste `stock-v1` pendant le scénario.

Critères de réussite :

- Premier GET : réponse cliente 200 et corps `stock=12` ; un appel d'origine et un rendu.
- Deuxième GET : la requête envoyée par le proxy à l'origine contient le validateur stocké ; l'origine renvoie 304 sans rendre le corps.
- Le client reçoit néanmoins 200 et `stock=12` ; total deux appels d'origine, un seul rendu. Capturer les octets envoyés par le proxy si sa réponse utilise un fichier interne.
- Dans une expérience séparée avec un store vide et `no-store`, deux GET causent deux appels d'origine et deux rendus ; aucune copie nouvellement stockée n'est réutilisée.
- Cas négatif : en changeant la version à `stock-v2` et le corps à `stock=9` dans un état réellement partagé par l'origine de test, l'ancien validateur ne provoque plus 304 ; le client reçoit le nouveau corps.

Livrable : trace des deux échanges HTTP, compteurs et nettoyage du store. Ne pas conclure « pas de cache » à partir du seul fait que l'origine a été appelée deux fois.

**Indice 1 :** un proxy peut posséder le corps complet alors que son client ne le possède pas.

**Indice 2 :** lire `HttpCache::validate()` : après un 304 d'origine, il met à jour l'entrée et peut renvoyer son corps stocké ; le statut du contrôleur n'est pas forcément le statut final.

## 14. Modification du détail, liste encore fraîche

**Niveau : avancé · Durée : 55–75 min · Prérequis : exercice 7 et [invalidation HTTP](approfondissement.md).**

Une origine de test expose `GET /lesson/cache/items/7`, `PUT` sur cette même URI et `GET /lesson/cache/items`. Les deux lectures ont une fraîcheur publique de 600 secondes. Utiliser un dépôt fixture en mémoire détenu par l'origine de test, stable pendant tout le scénario, sans Doctrine.

Données : titre initial `Carnet bleu`, version `1`. Le PUT valide remplace le titre par `Carnet vert`, version `2`, et répond 204 sans `Location` ni `Content-Location`. Le payload passe par un DTO mappé si l'origine est un contrôleur applicatif.

Critères de réussite, toutes les requêtes passant par le même proxy :

- Charger détail et liste puis les relire : un seul rendu d'origine par URI avant modification.
- Après le PUT réussi, relire le détail renvoie `Carnet vert` et incrémente son compteur ; l'ancienne entrée fraîche n'est plus réutilisée telle quelle.
- Relire la liste renvoie encore sa version stockée `Carnet bleu`, sans nouvel appel d'origine : documenter cette limite volontaire du scénario.
- Cas négatif indépendant, avec fixtures et store réinitialisés : un PUT refusé en 409 ne modifie rien ; une lecture suivante du détail frais garde son contenu et n'ajoute aucun rendu d'origine.
- Proposer une politique explicite pour la liste (par exemple expiration courte ou URL versionnée), avec justification de sa fenêtre de cohérence. Ne pas ajouter un faux en-tête Location uniquement pour forcer le résultat.

Livrable : tableau versions/compteurs/statuts et explication de l'URI invalidée. Aucun test ne doit attendre 600 secondes ; l'objectif est justement d'observer l'entrée encore fraîche.

**Indice 1 :** un proxy voit des URI et des réponses HTTP, pas les dépendances métier entre une fiche et toutes les listes possibles.

**Indice 2 :** lire le chemin `invalidate()` pour les méthodes non sûres et sa condition de statut ; garder ensuite la lecture de la liste séparée de celle du détail.
