# Exercices — Module 04

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Code dans `src/Http/` (namespace `App\Http`), tests dans `tests/Http/` (namespace `App\Tests\Http`) : `php bin/phpunit tests/Http`. Écris le test d'abord. Fil rouge : une mini-API d'« articles » (`/articles`).

## 1. Méthodes et sémantique
Écris `App\Http\MethodPolicy` avec `isSafe(string $method)`, `isIdempotent(string $method)` (sans utiliser `Request`), puis un test qui compare tes résultats à `Request::create('/', $method)->isMethodSafe()` / `isMethodIdempotent()` pour `GET, HEAD, OPTIONS, TRACE, PUT, DELETE, POST, PATCH, QUERY`.
- Questions : où ton implémentation diffère-t-elle de Symfony (indice : `PURGE`) ? Pourquoi `QUERY` est-elle sûre alors qu'elle porte un corps ?

## 2. Routes, 405 et `Allow`
Crée un controller `ArticleController` avec `list` (GET), `create` (POST), `delete` (DELETE) sur `/articles` et `/articles/{id}` (`requirements: ['id' => '\d+']`).
- Tests (`WebTestCase`) : `PUT /articles/1` → 405 avec un header `Allow` cohérent ; `GET /articles/abc` → 404 ; `HEAD /articles` → 200 sans corps mais mêmes headers que GET.
- Questions : qui produit le 405 (quel composant, quelle exception) ? Que se passe-t-il pour `OPTIONS` ?

## 3. Codes de statut et exceptions
Ajoute : `POST /articles` → **201** + header `Location` ; `GET /articles/{id}` inconnu → **404** via `NotFoundHttpException` ; `DELETE` → **204** sans corps ; un conflit de titre → **409** via `ConflictHttpException`.
- Tests : statut, `Location`, corps vide pour 204.
- Variante : crée `ArticleLockedException` annotée `#[WithHttpStatus(423)]` (ou 409) et vérifie que le code HTTP suit. Quelle différence avec implémenter `HttpExceptionInterface` ?

## 4. Redirections
Crée `/old-articles/{id}` qui redirige vers `/articles/{id}`.
- Tests : une version en 301 (ou 308), une version en 303 après un POST (Post/Redirect/Get).
- Questions : pourquoi utiliser 308 plutôt que 301 pour un POST ? Qu'est-ce que le navigateur refera avec 302 ?

## 5. Content negotiation
Crée `GET /articles/{id}` qui répond en JSON ou en HTML selon `Accept`, avec le paramètre de route `_format` prioritaire.
- Tests : `Accept: application/json` → JSON ; `Accept: text/html` → HTML ; `Accept: application/xml` → 406 ; `/articles/1.json` force le JSON même avec `Accept: text/html` ; header `Vary: Accept` présent.
- Questions : quel ordre utilise `getPreferredFormat()` ? Que fait `getRequestFormat()` par défaut ? Pourquoi le `Vary` est-il indispensable avec un cache ? 406 ou 415 : qui est concerné (réponse ou requête) ?

## 6. Cookies sûrs
Crée un service `PreferenceCookieFactory` qui fabrique un cookie `prefs` : expiration 30 jours, `HttpOnly`, `Secure`, `SameSite=Strict`, `Path=/`. Un controller le pose, un autre le lit et le supprime (`clearCookie`).
- Tests unitaires : `(string) $cookie` contient `httponly`, `secure`, `samesite=strict`, une date d'expiration correcte (utilise l'`Clock`/`MockClock` si tu veux contrôler le temps). Test fonctionnel : le `CookieJar` du client contient/ne contient plus le cookie.
- Questions : quelles sont les valeurs par défaut de `Cookie::create()` ? Pourquoi `SameSite=None` impose `Secure` ? Que fait `Cookie::setSecureDefault()` ?

## 7. `#[MapRequestPayload]` et validation
Crée `ArticleInput` (DTO avec `#[Assert\NotBlank]`, `#[Assert\Length]`) et utilise `#[MapRequestPayload]` dans `create`.
- Tests : JSON valide → 201 ; titre vide → 422 ; JSON malformé → 400 ; `Content-Type: text/plain` → 415 ; champ de mauvais type (`"views": "abc"`) → 422 ; corps vide → 422.
- Questions : que change `acceptFormat: 'json'` ? Quels composants doivent être installés (`debug:container serializer validator`) ? À quelle priorité le mapping a-t-il lieu (`debug:event-dispatcher kernel.controller_arguments`) ?

## 8. `#[MapQueryString]` / `#[MapQueryParameter]`
Ajoute `GET /articles?page=2&limit=10&sort=title` avec un DTO `ArticleFilter` (`#[MapQueryString]`) et un `#[MapQueryParameter] int $page = 1`.
- Tests : valeurs par défaut quand la query est vide ; `limit=0` invalide (contrainte `Range`) ; `page=abc` → quel code ? (observe le défaut puis change `validationFailedStatusCode` à 400 et teste).
- Réflexion : pourquoi Symfony choisit-il 404 par défaut pour la query et 422 pour le corps ? Dans quel cas utiliser `mapWhenEmpty: true` ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.


## Critères complémentaires des exercices 1–8

- **1–2** : distinguer la liste de la RFC 9110 et les extensions reconnues par Symfony (`QUERY`, `PURGE`). Pour 405, tester l'appartenance des méthodes à `Allow`, sans imposer un ordre textuel.
- **3–4** : générer `Location` avec le routeur ; désactiver le suivi automatique pour vérifier le premier statut de redirection, puis suivre et observer la méthode.
- **5** : ajouter deux valeurs `q` différentes ; tester le type réellement servi, pas seulement un mot dans le corps. Le cas XML doit être refusé même si Symfony connaît ce format.
- **6** : utiliser une requête HTTPS de test pour qu'un cookie `Secure` soit renvoyé ; conserver nom/path/domain à la suppression.
- **7–8** : tester aussi l'absence de champ et un corps vide ; annoncer les valeurs par défaut du DTO et le statut attendu. `mapWhenEmpty` est une variante **8.1**, distincte du socle 8.0.

## 9. Session et flash consommable (fondation)

Ajoute un POST qui enregistre une préférence en session et un flash, puis redirige en 303 vers une page de résultat.
- Attendu avec le même client : préférence conservée, flash affiché une fois ; au GET suivant, préférence encore présente mais flash absent. Un second client indépendant ne voit ni préférence ni flash.
- Variante : `peek()` deux fois conserve le message ; `get()` le consomme.
- Indice : cookie de session et donnée de session ne sont pas la même chose ; inspecter le `CookieJar` et les en-têtes du cache.

## 10. Upload sous contrat (application)

Ajoute un endpoint multipart acceptant un fichier texte, maximum 10 KiB, avec `#[MapUploadedFile]` et contraintes adaptées ; le service choisit un nom serveur dans un dossier temporaire de test hors `public/`.
- Attendu : fichier valide → 201 ; absent ou trop gros → 422 selon le contrat choisi ; contenu non autorisé → 422 ; aucun nom fourni par le client ne permet d'écrire hors du dossier. Le test nettoie ses fichiers.
- Preuve : `UploadedFile` en mode test, appel HTTP multipart, existence et contenu du fichier stocké ; aucun déplacement sur un chemin construit depuis le nom d'origine.
- Indice : voir `MapUploadedFile`, `Assert\File` et les méthodes de `UploadedFile` ; ne pas faire confiance au MIME annoncé.

## 11. Téléchargement et HEAD (diagnostic)

Expose un fichier fixture via `BinaryFileResponse` ou le helper `file()`, nom public `rapport.txt`.
- Attendu : GET → 200, octets de la fixture, `Content-Disposition` avec nom public ; HEAD → 200 et aucun octet de corps ; fichier inconnu → 404 ; aucun chemin local révélé dans les erreurs.
- Variante : `Range: bytes=0-3` sur un fichier régulier → 206 et quatre octets ; observer `Content-Range`.
- Indice : tester l'envoi réel ou le streaming capturé ; `getContent()` n'est pas une lecture universelle des réponses fichier/stream.

## 12. Négociation et contrat d'erreur (approfondissement)

Une route propose HTML et JSON ; stabilise ses erreurs JSON en champs publics `code` et `message`.
- Attendu : JSON de qualité supérieure choisi ; `_format` explicite respecté ; représentation non proposée → 406 ; corps de requête non supporté → 415 ; payload invalide → 422 ; en-tête `Vary: Accept` sur les réponses négociées.
- Preuve : assertions sur statut, Content-Type et structure, absence de stack trace/chemin interne dans un environnement sans debug.
- Indice : négocier les représentations réellement offertes, pas tous les formats connus de `Request`.

## Défi final — parcours d'articles et pièces jointes

Sans Doctrine, utilise des fixtures et un service en mémoire contrôlé par les tests. Crée un article valide, observe 201/Location, affiche sa représentation négociée, associe une pièce jointe, puis la télécharge en GET et HEAD. Couvre séparément 400, 404, 405 avec Allow, 406, 415 et 422 ; un flash de confirmation ne doit apparaître qu'une fois. Précise comment l'état du scénario est conservé entre requêtes de test, sans supposer qu'un Kernel redémarré conserve un tableau en mémoire.

## 13. Reprendre un téléchargement sans mélanger les versions

**Niveau : intermédiaire · Durée : 40–55 min · Prérequis : exercice 11 et [plages HTTP](approfondissement.md).**

Une route `/lesson/download/archive` sert un fichier régulier de dix octets, exactement `abcdefghij`, avec `BinaryFileResponse` et un ETag fort `archive-v3`. Le fichier est une fixture temporaire hors `public/`, créée et supprimée par le test.

Critères de réussite par requêtes HTTP et capture réelle du corps :

- GET sans condition → 200 et dix octets ; `Range: bytes=2-5` → 206, corps `cdef`, `Content-Length: 4`, `Content-Range: bytes 2-5/10`.
- `Range: bytes=-3` → 206 et `hij` ; aucune interprétation de caractères multioctets n'entre dans ce jeu ASCII.
- Même plage avec `If-Range: "archive-v3"` → 206 ; avec `If-Range: "archive-v2"` → 200 et fichier complet.
- Cas négatif : `Range: bytes=20-30` → 416 avec `Content-Range: bytes */10`, sans octets du fichier.
- HEAD → aucun corps, et une seconde requête GET normale retourne toujours les dix octets ; chaque appel utilise une réponse neuve.

Livrable : tableau requête/statut/en-têtes/taille/corps. Ne pas écrire de parseur de Range : vérifier le comportement du composant et limiter l'exercice à ces plages uniques.

**Indice 1 :** la préparation d'une réponse fichier dépend de la requête, et ses bornes de plage sont inclusives.

**Indice 2 :** inspecter `BinaryFileResponse::prepare()` et `hasValidIfRangeHeader()` ; un validateur If-Range différent conduit à ignorer la plage, pas à renvoyer 304.

## 14. Retrouver l'origine publique derrière un proxy

**Niveau : avancé · Durée : 45–60 min · Prérequis : objets Request, tests de service et [proxies de confiance](approfondissement.md).**

Créer un service `RequestOriginInspector` qui reçoit une `Request` lors de son appel et retourne protocole, hôte et adresse client à partir des méthodes HttpFoundation. Il doit fonctionner sans conserver la requête entre deux appels.

Données déterministes : URI interne `http://app.internal/lesson/origin`, `REMOTE_ADDR=192.0.2.10`, `X-Forwarded-Proto=https`, `X-Forwarded-For=198.51.100.23`, `X-Forwarded-Host=public.example`. Ces adresses sont des fixtures de documentation.

Critères de réussite dans des tests de service appelant réellement les méthodes de `Request` :

- Sans proxy approuvé : protocole `http`, hôte `app.internal`, IP `192.0.2.10`.
- En approuvant seulement ce proxy et les catégories FOR/PROTO : protocole `https`, IP `198.51.100.23`, hôte toujours `app.internal`.
- En autorisant aussi la catégorie HOST : hôte `public.example` ; justifier ce changement de confiance.
- Cas négatif : avec `REMOTE_ADDR=203.0.113.9` non approuvée et les mêmes en-têtes, le protocole reste `http` et les informations transmises ne deviennent pas fiables.
- Avec `trusted_hosts` limité à `public.example`, une requête effectivement évaluée avec un autre host est refusée par HttpFoundation. Distinguer l'exception de composant du 400 produit dans un cycle HTTP complet.

Isoler chaque test de service `KernelTestCase` avec les attributs PHPUnit `RunInSeparateProcess` et `PreserveGlobalState(false)`. Démarrer le kernel et récupérer le service d'abord, puis définir explicitement proxies, catégories d'en-têtes et hosts avant de construire la requête. Ne pas redémarrer le kernel après ce réglage : il réappliquerait sa configuration. Aucun réglage réel du serveur n'est modifié.

Livrable : matrice des quatre origines calculées et tests de refus. Utiliser les API Symfony ; lire directement `X-Forwarded-For` dans le service contournerait le mécanisme étudié.

**Indice 1 :** l'adresse de l'émetteur et les catégories d'en-têtes sont deux paramètres indépendants.

**Indice 2 :** ces réglages sont statiques, mais leurs getters ne sont pas tous symétriques : `getTrustedHosts()` renvoie des expressions déjà délimitées, que `setTrustedHosts()` délimiterait à nouveau. L'isolation de processus évite cette restauration incorrecte ; si tu restaures manuellement des hosts, conserve les motifs bruts d'origine.
