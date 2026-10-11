# Module 04 — HTTP (RFC 9110)

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

> **Périmètre** : application vérifiée en **Symfony 8.1.8**, PHP ≥ 8.4 ; la [certification Symfony 8](https://certification.symfony.com/exams/symfony.html) vise **Symfony 8.0**. Les nouveautés marquées **8.1** sont un complément de pratique, pas des exigences de cet examen. Suivre la [méthode de travail du parcours](../README.md) : prédire, observer, tester, expliquer.

## Objectifs et prérequis

Prérequis : [Kernel](../02-kernel-architecture/README.md), contrôleurs et tests HTTP. Savoir relier une méthode, des en-têtes et un corps à un contrat observable, puis traiter session, upload et téléchargement sans exposer des données internes. Le stockage des articles reste en mémoire : Doctrine n'est pas installé. Les [routes avancées](../11-routing/README.md) et la [validation approfondie](../12-validator/README.md) ont leurs propres modules.

La série RFC 9110/9111/9112 remplace les RFC 7230-7235 ; la RFC 9110 porte sur la sémantique (dont la 7231 sur la sémantique, encore citée dans les docblocks Symfony). Ce module relie la sémantique HTTP aux objets HttpFoundation et aux resolvers de HttpKernel. Vérifié sur Symfony 8.1.

## 1. Méthodes : sûres, idempotentes, cacheables

| Méthode | Sûre | Idempotente | Cacheable (RFC) | Remarque |
|---|---|---|---|---|
| GET | oui | oui | oui | |
| HEAD | oui | oui | oui | comme GET, sans corps |
| OPTIONS | oui | oui | non | découverte (`Allow`), CORS preflight |
| TRACE | oui | oui | non | |
| PUT | non | oui | non | remplace la ressource |
| DELETE | non | oui | non | |
| POST | non | **non** | rarement | création, traitement |
| PATCH | non | **non** (en général) | non | modification partielle |
| CONNECT | non | non | non | tunnel |
| QUERY (extension, hors RFC 9110) | oui | oui | oui | corps de requête + sémantique de lecture ; supporté depuis Symfony 7.4 |

- **Sûre** = n'a pas d'effet de bord attendu côté serveur (lecture seule). **Idempotente** = N requêtes identiques ont le même effet qu'une seule.
- Une méthode sûre est idempotente, pas l'inverse (PUT, DELETE).
- Symfony (`Request`, vérifié dans le code) :
  - `isMethodSafe()` : `GET, HEAD, OPTIONS, TRACE, QUERY`
  - `isMethodIdempotent()` : `HEAD, GET, PUT, DELETE, TRACE, OPTIONS, PURGE, QUERY`
  - `isMethodCacheable()` : `GET, HEAD, QUERY`
  - Constantes `Request::METHOD_*` (dont `METHOD_QUERY`, `METHOD_PURGE`).
- Routing : `#[Route('/x', methods: ['GET', 'HEAD'])]` ; une méthode non autorisée donne **405** + header `Allow` (`MethodNotAllowedHttpException`).
- Override de méthode (formulaires HTML) : `framework.http_method_override` (défaut `false`) lit `_method` sur un POST ; `allowed_http_method_override` restreint la liste. **8.0** : l'override vers `GET`, `HEAD`, `CONNECT`, `TRACE` est **ignoré**. Le header `X-HTTP-METHOD-OVERRIDE` est lu par `Request::getMethod()` sur un POST (voir le code).
- `Request::createFromGlobals()` parse le corps des requêtes `PUT`, `DELETE`, `PATCH`, `QUERY` (depuis 7.4).

## 2. Codes de statut

| Classe | Exemples à connaître (constante `Response::HTTP_*`) |
|---|---|
| 1xx | 100 Continue, 103 Early Hints (`HTTP_EARLY_HINTS`) |
| 2xx | 200 OK, 201 Created (+ `Location`), 202 Accepted, 204 No Content, 206 Partial Content |
| 3xx | 301 Moved Permanently, 302 Found, 303 See Other, 304 Not Modified, 307 Temporary Redirect, 308 Permanent Redirect |
| 4xx | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 405 Method Not Allowed, 406 Not Acceptable, 409 Conflict, 410 Gone, 412 Precondition Failed, 413 Content Too Large, 415 Unsupported Media Type, 422 Unprocessable Content, 428 Precondition Required, 429 Too Many Requests |
| 5xx | 500 Internal Server Error, 501, 502, 503 Service Unavailable (+ `Retry-After`), 504 |

Distinctions fréquentes :
- **401** : non authentifié (header `WWW-Authenticate`) ; **403** : accès refusé ; ce statut ne prouve pas à lui seul que le client est authentifié.
- **301/302 vs 307/308** : 301/302 peuvent faire passer un POST en GET ; 307/308 **conservent** la méthode. 303 = « va faire un GET ailleurs » (pattern Post/Redirect/Get). 308 = permanent + méthode conservée.
- **400** syntaxe/format invalide ; **422** syntaxiquement correct mais sémantiquement invalide (validation) ; **409** conflit d'état.
- **204** : pas de corps ; **304** : pas de corps, validation de cache réussie (module 05).
- **Constantes** : `Response::HTTP_UNPROCESSABLE_ENTITY` vaut 422 (texte `Unprocessable Content`) ; `HTTP_REQUEST_ENTITY_TOO_LARGE` = 413 (texte `Content Too Large` dans les `statusTexts`).
- Exceptions : `HttpExceptionInterface` (`getStatusCode()`, `getHeaders()`) ; classes dans `Symfony\Component\HttpKernel\Exception\` (`NotFoundHttpException`, `AccessDeniedHttpException`, `ConflictHttpException`, `UnprocessableEntityHttpException`, `TooManyRequestsHttpException`, `MethodNotAllowedHttpException`, `UnsupportedMediaTypeHttpException`, `NotAcceptableHttpException`...). `HttpException::fromStatusCode($code)` pour un code arbitraire.
- `#[WithHttpStatus(code, headers)]` sur une exception métier → code HTTP sans implémenter l'interface ; `#[WithLogLevel]` pour le niveau de log (voir `vendor/symfony/http-kernel/Attribute/`).
- Helpers sur `Response` : `isSuccessful()`, `isRedirection()`, `isClientError()`, `isServerError()`, `isNotFound()`, `isEmpty()`, `isRedirect($location)`.

## 3. Content negotiation

Négociation proactive : le client envoie `Accept`, `Accept-Language`, `Accept-Encoding`, `Accept-Charset` avec des qualités `q` (`Accept: application/json;q=0.9, text/html`).

```php
$request->getAcceptableContentTypes();   // types triés par qualité
$request->getPreferredFormat('html');    // 'json', 'html'... : _format d'abord, puis Accept
$request->getPreferredLanguage(['fr', 'en']);
$request->getLanguages(); $request->getEncodings(); $request->getCharsets();
$request->getRequestFormat();            // _format de la route sinon 'html' par défaut
$request->getContentTypeFormat();        // format du Content-Type du corps (json, xml, form...)
$request->getFormat('application/json'); // 'json' ; getMimeType('json')
```

- `getPreferredFormat()` : le docblock précise qu'il faut alors envoyer **`Vary: Accept`** (sinon un cache intermédiaire peut servir la mauvaise représentation).
- Statuts de négociation : **406 Not Acceptable** (aucune représentation acceptable côté réponse), **415 Unsupported Media Type** (le `Content-Type` du corps n'est pas supporté côté requête).
- Route : `#[Route('/post.{_format}', requirements: ['_format' => 'json|html'], defaults: ['_format' => 'html'])]`, ou `format: 'json'` dans `#[Route]`.
- `Request::setFormat('foo', ['application/foo'])` ajoute un format ; `AddRequestFormatsListener` (priorité 100 sur `kernel.request`) enregistre des formats additionnels.
- Négociation côté **requête** : `Content-Type` ; `Accept` pour la **réponse**. Ne pas confondre.

## 4. Cookies (HttpFoundation)

```php
$cookie = Cookie::create('token')
    ->withValue('abc')
    ->withExpires(new \DateTimeImmutable('+1 hour'))
    ->withPath('/')
    ->withSecure(true)
    ->withHttpOnly(true)
    ->withSameSite(Cookie::SAMESITE_STRICT)
    ->withPartitioned();                       // CHIPS (cookies partitionnés)
$response->headers->setCookie($cookie);
$response->headers->clearCookie('token', '/', null, true, true, Cookie::SAMESITE_LAX);
```

Valeurs par défaut lues dans le code de `Cookie::__construct()` : `expire = 0` (cookie de session), `path = '/'`, `domain = null`, `secure = null` (= « auto » : HTTPS seulement si la requête l'est ; `Cookie::setSecureDefault()`), `httpOnly = true`, `sameSite = 'lax'`, `partitioned = false`, `raw = false`.

- **HttpOnly** : inaccessible à JavaScript (réduit l'impact d'un XSS).
- **Secure** : envoyé seulement en HTTPS.
- **SameSite** : `Lax` (envoyé sur navigation top-level GET cross-site), `Strict` (jamais cross-site), `None` (toujours — les navigateurs exigent alors `Secure`).
- `Cookie::SAMESITE_LAX|STRICT|NONE` ; une autre valeur lève `InvalidArgumentException`.
- Session : `framework.session.cookie_secure` (`auto` par défaut), `cookie_httponly` (`true`), `cookie_samesite` (`lax`).
- Lire : `$request->cookies->get('name')`. Un cookie est supprimé en envoyant le même nom/path/domain avec expiration passée (`clearCookie`).
- Préfixes `__Host-`/`__Secure-` : conventions navigateur (Secure ; `__Host-` : path `/`, pas de Domain).

## 5. Mapper la requête : attributs de controller

| Attribut | Source | Échec : code par défaut |
|---|---|---|
| `#[MapQueryParameter]` | un paramètre de query (`filter_var`, types scalaires, enum, array) | **404** |
| `#[MapQueryString]` | toute la query string → DTO (denormalize) | **404** |
| `#[MapRequestPayload]` | corps (json, xml, form...) → DTO ou `array` typé | **422** |
| `#[MapRequestHeader]` (8.1) | un header → argument | 400 |
| `#[MapUploadedFile]` | fichier(s) uploadés (+ `constraints`) | 422 |
| `#[MapDateTime]` | date depuis un paramètre/attribut avec `format` | — |

```php
#[Route('/articles', methods: ['POST'])]
public function create(
    #[MapRequestPayload(
        acceptFormat: 'json',
        validationGroups: ['create'],
        validationFailedStatusCode: Response::HTTP_UNPROCESSABLE_ENTITY,
    )] ArticleInput $input,
): Response { /* ... */ }

#[Route('/articles', methods: ['GET'])]
public function list(
    #[MapQueryString(validationFailedStatusCode: 400)] ?ArticleFilter $filter = null,
    #[MapQueryParameter] int $page = 1,
): Response { /* ... */ }
```

Comportement lu dans `RequestPayloadValueResolver` :
- Le resolver est enregistré à la fois comme `ValueResolver` (module 02) et comme listener `kernel.controller_arguments` à la priorité **-10100** (après `ControllerAttributesListener` à -10000 : les attributs de contrôle d'accès passent avant).
- `MapRequestPayload` : le format vient du `Content-Type` (`getContentTypeFormat()`) ; absent ou non supporté → **415** (`UnsupportedMediaTypeHttpException`) ; `acceptFormat` restreint les formats autorisés (sinon 415) ; JSON/XML invalide → **400** (`BadRequestHttpException`).
- Erreurs de type (ex. string pour un int) : collectées via `collect_denormalization_errors` et transformées en violations (ce qui donne le code `validationFailedStatusCode`).
- Violations de contraintes (`#[Assert\...]` du DTO) : `HttpException` avec `ValidationFailedException` en `previous` ; le message liste les violations.
- Corps vide : si l'argument est nullable ou a une valeur par défaut → `null`/défaut (sauf `mapWhenEmpty: true`) ; sinon `HttpException` avec le code d'échec.
- Nécessite `symfony/serializer` (et `symfony/validator` pour la validation). Sans validator, seules les erreurs de dénormalisation sont gérées.
- 8.1 : `validationGroups` accepte `Expression`, `Closure` ou `GroupSequence` ; arguments **variadiques** supportés (`MapRequestPayload`) ; `UploadedFile` supporté dans le payload ; `$type` pour mapper un tableau d'objets (`array` + `type:`).

## 6. Validation

```php
final readonly class ArticleInput
{
    public function __construct(
        #[Assert\NotBlank, Assert\Length(max: 120)] public string $title = '',
        #[Assert\Range(min: 0)] public int $views = 0,
    ) {}
}
```
- Le validateur lit les attributs `#[Assert\...]` (annotations Doctrine supprimées).
- Pour retourner une erreur JSON structurée, soit tu attrapes `HttpException` avec `ValidationFailedException` en `previous` (listener `kernel.exception`), soit tu laisses le rendu d'erreur par défaut (page d'erreur ou, pour un format `json`, le Serializer s'il est disponible).
- Groupes : `groups: ['create']` sur la contrainte + `validationGroups` sur l'attribut.

## Ce qui change vs Symfony 6/7

- 8.0 : l'override de méthode vers GET/HEAD/CONNECT/TRACE est ignoré ; `Request::get()` **supprimé** (utiliser `attributes`, `query`, `request`).
- 7.4+ : méthode `QUERY` supportée (safe, idempotente, cacheable) ; `Request::createFromGlobals()` parse le corps de PUT/DELETE/PATCH/QUERY.
- 6.3 : `#[MapQueryParameter]`, `#[MapQueryString]`, `#[MapRequestPayload]` ; 7.1 : `#[MapUploadedFile]` (d'après le CHANGELOG de HttpKernel).
- 8.1 : `#[MapRequestHeader]`, validation des paramètres de route typés avant le controller, `mapWhenEmpty`, `validationGroups` dynamiques, variadiques.
- 8.1 : `ParameterBag::getInt()`/`getBoolean()` lèvent `UnexpectedValueException` quand la conversion est impossible (au lieu de `0`/`false`). Assigner directement des propriétés publiques de `Request`/`Response` est déprécié.
- 8.0 : `ResponseHeaderBag::clearCookie()` accepte `$partitioned`.
- Constantes de statut : les textes ont suivi la RFC 9110 (« Content Too Large », « Unprocessable Content »).

## Piège certification
- Idempotent ≠ sûr : `PUT` et `DELETE` sont idempotents mais pas sûrs. `POST` n'est ni l'un ni l'autre.
- 401 vs 403 ; 301/302 vs 307/308 (méthode conservée) ; 400 vs 422 ; 406 vs 415.
- Par défaut `MapQueryString` / `MapQueryParameter` échouent en **404**, `MapRequestPayload` en **422** : à connaître.
- `HEAD` : mêmes en-têtes que `GET`, **sans corps** ; Symfony supprime le corps via `Response::prepare()`.
- `getPreferredFormat()` ⇒ penser à `Vary: Accept`.
- `SameSite=None` ⇒ `Secure` nécessaire ; `HttpOnly` n'empêche pas CSRF, c'est `SameSite`/jeton CSRF qui s'en charge.
- Cookie par défaut Symfony : `SameSite=Lax`, `HttpOnly=true`, `Secure` automatique (selon HTTPS).
- `Request::getMethod()` retourne la méthode **overridée**, `getRealMethod()` la vraie.

## Observer dans l'app

Le profiler n'est **pas installé** dans ce dépôt. Les commandes, traces et tests fonctionnent sans lui ; les observations via `/_profiler` ou `enableProfiler()` ci-dessous sont optionnelles, après installation volontaire avec `composer require --dev symfony/profiler-pack` (Flex). Commencer par `var/log/dev.log` lorsqu'il existe.
- `bin/console debug:router` (colonne *Method*), `debug:router <nom> --show-aliases`.
- `curl -i -X OPTIONS http://localhost:8000/...` ; `curl -i -H 'Accept: application/json' ...` ; `curl -I` (HEAD) ; `curl -i -b/-c` pour les cookies. (Serveur : `symfony serve -d`.)
- Profiler : onglets **Request** (méthode, format, attributs `_route`, headers, cookies), **Exception**, **Validator**, **Serializer** (si activé).
- `bin/console debug:config framework http_method_override`, `debug:config framework session`.
- `bin/console debug:container --tag=controller.argument_value_resolver` et `debug:event-dispatcher kernel.controller_arguments` (voir le resolver à -10100).
- Tests : `WebTestCase` + `$client->request('POST', '/x', server: ['CONTENT_TYPE' => 'application/json'], content: '{...}')` ; `assertResponseStatusCodeSame(422)`, `assertResponseHeaderSame()`, `assertResponseCookieValueSame()`, `$client->getCookieJar()` ; test unitaire de `Request::create()`, `Cookie::create()`, `(string) $cookie`.

## Doc officielle
- https://www.rfc-editor.org/rfc/rfc9110 (sémantique HTTP)
- https://symfony.com/doc/current/components/http_foundation.html
- https://symfony.com/doc/current/controller.html#mapping-request-data-to-typed-objects
- https://symfony.com/doc/current/routing.html
- https://symfony.com/doc/current/validation.html
- https://symfony.com/doc/current/reference/configuration/framework.html (`http_method_override`, session)
- https://symfony.com/doc/current/controller/error_pages.html


## 7. Session et flashes : suivre deux requêtes

Le cookie de session transporte un identifiant ; les données sont gérées côté serveur par le stockage configuré. Injecter `RequestStack` dans un service et demander la session au moment de l'opération ; dans un contrôleur HTTP, `$request->getSession()` convient. La session démarre à l'utilisation, pas parce qu'un type `RequestStack` a été injecté.

```php
$session = $request->getSession();
$session->set('display_mode', 'compact');
$session->getFlashBag()->add('notice', 'Préférence enregistrée');
// Dans la requête suivante :
$messages = $session->getFlashBag()->get('notice'); // lecture consommatrice
```

`peek()` observe sans consommer ; `get()` retire les messages du type demandé, `all()` consomme tous les types. Un flash est donc « lu une fois », pas simplement « valable un nombre fixe de secondes ». Utiliser le même client BrowserKit après la redirection pour conserver le cookie ; un second client indépendant ne doit pas voir la préférence du premier. L'accès à la session affecte aussi le [cache HTTP](../05-cache-http/README.md).

## 8. Upload et téléchargement : deux contrats opposés

Une requête multipart fournit des `UploadedFile` dans `$request->files`. `#[MapUploadedFile]` permet de recevoir et valider le fichier ; les règles de taille et de type relèvent de Validator. Le nom, l'extension et le MIME **déclarés par le client** ne sont pas fiables. Le service de stockage choisit un nom serveur, valide le contenu et place le fichier dans un répertoire contrôlé hors `public/` si son accès est privé. Le cas d'erreur PHP d'upload doit aussi être traité : `isValid()` n'est pas garanti parce qu'un objet existe.

Pour télécharger un fichier existant, `BinaryFileResponse` gère le transfert et les requêtes de plage. Le helper `AbstractController::file()` permet de définir le nom de téléchargement sans livrer au client le chemin de stockage.

```php
// Extrait de contrôleur ; chemin choisi côté serveur, jamais concaténé depuis l'URL.
return $this->file($absolutePath, 'rapport.pdf');
```

`Content-Disposition: attachment` suggère le téléchargement ; `inline` demande l'affichage. Le serveur doit encore contrôler l'accès. `StreamedResponse` convient à une production progressive (CSV, export) : le callback s'exécute à l'envoi, donc `getContent()` ne contient pas l'export. Observer les octets envoyés ou capturer explicitement la sortie dans le test adapté.

## 9. HEAD, négociation et erreurs : préciser les garanties

`HEAD` ne doit pas envoyer de corps ; cela ne prouve pas que le contrôleur n'a pas calculé le contenu. `Response::prepare($request)` supprime le contenu d'une réponse HEAD. Certains en-têtes coûteux à calculer peuvent être omis selon la sémantique HTTP : comparer les en-têtes stables pertinents, pas `Date` octet pour octet.

`getPreferredFormat()` aide à choisir un format connu, mais ne garantit pas à lui seul qu'il figure dans les formats **offerts par l'application**. Si la route ne sert que HTML/JSON, définir cette liste, traiter le cas sans représentation acceptable et éviter qu'un défaut `'html'` masque un 406. Tester `Accept: application/json;q=0.9,text/html;q=0.5`, un type non supporté et un `_format` imposé. `Content-Type` décrit le corps reçu ; le changer ne demande pas une réponse dans ce format.

Une API doit stabiliser ses erreurs : statut, type de contenu et champs publics (par exemple `code`, `message`, violations). En production, aucun chemin local, secret ou stack trace ne doit apparaître. Un test fonctionnel porte sur ce contrat public, pas sur le HTML du gestionnaire d'erreurs de développement. Approfondir [Validator](../12-validator/README.md) pour les violations et [Routing](../11-routing/README.md) pour priorité, host et génération d'URL.

Sources locales : `HttpFoundation/Session/Flash/FlashBag.php`, `File/UploadedFile.php`, `BinaryFileResponse.php`, `Response.php` ; tous ces chemins sont sous `vendor/symfony/` avec leur composant.
