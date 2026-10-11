# Module 05 — Cache HTTP

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

> **Périmètre** : application vérifiée en **Symfony 8.1.8**, PHP ≥ 8.4 ; la [certification Symfony 8](https://certification.symfony.com/exams/symfony.html) vise **Symfony 8.0**. Les nouveautés marquées **8.1** sont un complément de pratique, pas des exigences de cet examen. Suivre la [méthode de travail du parcours](../README.md) : prédire, observer, tester, expliquer.

## Objectifs et prérequis

Prérequis : [HTTP](../04-http-rfc9110/README.md), tests avec plusieurs requêtes, lecture d'en-têtes. Savoir distinguer stockage, fraîcheur et revalidation, calculer un TTL restant et empêcher qu'un cache partagé diffuse une réponse personnelle. Le [cache applicatif](../16-cache-applicatif/README.md) est distinct. **ESI est un enrichissement hors certification**, y compris ses exercices et questions conservés ci-dessous.

Le cache HTTP (RFC 9111, complète le module 04 sur RFC 9110) permet d'éviter de recalculer ou de re-télécharger une réponse. Deux modèles : **expiration** (« tu n'as pas besoin de me redemander ») et **validation** (« redemande-moi, je dirai si ça a changé »). Tout est installé : `symfony/http-foundation` (Response), `symfony/http-kernel` (`#[Cache]`, `HttpCache`, ESI), `symfony/framework-bundle` (config).

Deux types de caches :
- **privé** (navigateur) : propre à un utilisateur ;
- **partagé** (reverse proxy : Symfony HttpCache, Varnish, CDN) : sert plusieurs utilisateurs.

## 1. Expiration

```php
$response = new Response($content);
$response->setPublic();          // cachable par les caches partagés
$response->setMaxAge(600);       // Cache-Control: max-age=600      (tous les caches)
$response->setSharedMaxAge(3600);// Cache-Control: s-maxage=3600    (caches partagés seulement, prioritaire sur max-age pour eux)
```

- `max-age` : durée de fraîcheur en secondes. `Expires` (date HTTP) est l'ancienne variante ; `max-age` l'emporte si les deux sont présents.
- `s-maxage` : ne concerne que les caches **partagés** ; le navigateur l'ignore.
- `setTtl()` / `setClientTtl()` : raccourcis calculés relativement à l'âge/`s-maxage` (voir `Response.php`).
- `public` vs `private` : une réponse `private` n'est **pas** stockée par un cache partagé (`Response::isCacheable()` renvoie `false`, vérifié dans `Response.php`).
- `no-store` : interdit tout stockage. `no-cache` : stockage permis mais **revalidation obligatoire** à chaque usage. `must-revalidate` : une fois périmée, interdiction de servir sans revalider.
- Extensions RFC 5861 : `stale-while-revalidate` et `stale-if-error` (`setStaleWhileRevalidate()`, `setStaleIfError()`).
- `immutable` : `setImmutable()`.

Raccourci groupé : `$response->setCache(['public' => true, 'max_age' => 600, 's_maxage' => 3600, 'etag' => '...', 'last_modified' => $date])`. Clés autorisées (sinon `InvalidArgumentException`) : `must_revalidate, no_cache, no_store, no_transform, public, private, proxy_revalidate, max_age, s_maxage, stale_if_error, stale_while_revalidate, immutable, last_modified, etag`.

## 2. Validation (ETag, Last-Modified, 304)

Les extraits avec `Article` supposent un objet fourni par un service en mémoire ou un resolver explicite : le type seul ne charge aucune entité, Doctrine étant absent. Les expressions de `#[Cache]` nécessitent `symfony/expression-language`, absent actuellement ; privilégier ici l'API `Response`.

```php
#[Route('/article/{id}')]
public function show(Article $article, Request $request): Response
{
    $response = new Response();
    $response->setEtag(md5($article->getVersion()));     // ou setEtag($x, weak: true) => W/"..."
    $response->setLastModified($article->getUpdatedAt());
    $response->setPublic();

    if ($response->isNotModified($request)) {
        return $response;                                // 304, corps vide : on économise le rendu
    }

    return $response->setContent($this->renderView(...));
}
```

- Le client renvoie `If-None-Match` (ETag) et/ou `If-Modified-Since` ; `Response::isNotModified($request)` compare et appelle `setNotModified()` (statut 304, contenu supprimé).
- **Gain réel** seulement si tu testes `isNotModified()` **avant** de calculer le contenu coûteux.
- Les validateurs ne remplacent pas l'expiration : sans fraîcheur explicite, certains caches peuvent calculer une fraîcheur heuristique ; `no-cache` impose une revalidation avant réutilisation.
- ETag **faible** (`W/`) : équivalence sémantique ; **forte** : octet par octet.
- Requêtes concernées : `GET`/`HEAD` et, dans la version installée, `QUERY` (le 304 n'a de sens que pour les méthodes cachables, cf. `Request::isMethodCacheable()`). `If-Match`/`412` relèvent de la concurrence optimiste, pas du cache.

## 3. `Vary`

`Vary: Accept-Language` dit au cache : « la réponse dépend de cet en-tête de requête ; une entrée de cache par valeur ».

```php
$response->setVary(['Accept-Language']);          // remplace (replace: true par défaut)
$response->setVary('Accept-Encoding', replace: false);
```

- `Vary: Cookie` multiplie les variantes et réduit souvent fortement le taux de hit ; `Vary: *` empêche la réutilisation normale d'une réponse stockée.
- Une clé de variation absente de `Vary` = risque de servir la mauvaise version (ex. langue) à un autre utilisateur.

## 4. L'attribut `#[Cache]`

Dans `Symfony\Component\HttpKernel\Attribute\Cache` (plus dans `sensio/framework-extra-bundle` depuis Symfony 6.2). Applicable à une **classe, méthode, fonction**, répétable.

```php
use Symfony\Component\HttpKernel\Attribute\Cache;

#[Cache(maxage: 60, smaxage: 3600, public: true, vary: ['Accept-Language'])]
public function list(): Response { ... }

#[Cache(etag: 'article.getVersion()', lastModified: 'article.getUpdatedAt()', public: true)]
public function show(Article $article): Response { ... }   // expressions ExpressionLanguage
```

Paramètres du constructeur (vérifiés dans `Attribute/Cache.php`) : `expires`, `maxage`, `smaxage`, `public`, `mustRevalidate`, `vary`, `lastModified`, `etag`, `maxStale`, `staleWhileRevalidate`, `staleIfError`, `noStore`, `if`.

- `maxage`/`smaxage`/`maxStale`… acceptent un entier ou une chaîne (« 1 hour », durée relative).
- `lastModified` et `etag` acceptent des **expressions** (variables = arguments du contrôleur) ; `request` et `args` sont aussi disponibles en 8.1 ; une `\Closure` est possible (nouveau 8.1).
- Les attributs `etag`/`lastModified` déclenchent la validation **avant** l'exécution du contrôleur (le contrôleur n'est pas appelé si 304) : c'est l'écouteur `CacheAttributeListener` (en 8.0 : `kernel.controller_arguments` / `kernel.response` ; en **8.1**, événements d'attribut `kernel.controller_arguments.<FQCN de Cache>` / `kernel.response.<FQCN de Cache>`).
- `if:` (nouveau 8.1) : condition d'application de l'attribut (booléen, expression ou closure).

## 5. Reverse proxy Symfony : `HttpCache`

Implémenté en PHP (`HttpKernel\HttpCache\HttpCache`) : utile en dev/hébergement mutualisé, imite Varnish.

```yaml
# config/packages/framework.yaml
framework:
    http_cache: true          # ou :
    # http_cache:
    #     default_ttl: 0
    #     private_headers: ['Authorization', 'Cookie']
    #     allow_reload: false
    #     allow_revalidate: false
    #     stale_while_revalidate: 2
    #     stale_if_error: 60
    #     trace_level: short  # none | short | full
    #     trace_header: X-Symfony-Cache
```

Valeurs par défaut de `HttpCache` (vérifiées dans le code) : `debug=false`, `default_ttl=0`, `private_headers=['Authorization','Cookie']`, `skip_response_headers=['Set-Cookie']`, `allow_reload=false`, `allow_revalidate=false`, `stale_while_revalidate=2`, `stale_if_error=60`, `trace_level=none`.

- `private_headers` : une requête portant `Authorization` ou `Cookie` sans `Cache-Control: public` explicite côté réponse est traitée comme privée (non cachée).
- En-tête de diagnostic `X-Symfony-Cache` (avec `trace_level`/debug) : `GET /: miss, store` puis `fresh`, `stale`, `invalid`...
- Alternative sans `framework.http_cache` : envelopper le Kernel dans `public/index.php` (`new HttpCache($kernel, new Store($cacheDir))`).
- Invalidation : `HttpCache` ne fournit pas de PURGE natif simple → on joue sur l'expiration ou on étend la classe / on utilise Varnish ou `FOSHttpCacheBundle`.
- Le store par défaut : `Store` sur disque (`StoreInterface`). Les options doivent être comparées à `debug:config framework http_cache` : les valeurs de construction du composant ne prouvent pas toutes les valeurs finales du FrameworkBundle.

## 6. ESI (Edge Side Includes) — enrichissement hors certification

Permet de cacher une page **par fragments** de durées de vie différentes : la page principale est longue durée, un bloc (panier, user) est court ou privé.

```yaml
framework:
    esi: true
    fragments: { path: /_fragment }   # route des sous-requêtes signées
```

```twig
{{ render_esi(controller('App\\Controller\\SidebarController::index')) }}
{# render_esi / render_ssi / render_hinclude : fonction Twig render_* (HttpKernelExtension) #}
```

- Le reverse proxy annonce sa capacité avec `Surrogate-Capability: abc="ESI/1.0"` ; l'application répond avec `Surrogate-Control: content="ESI/1.0"` et insère `<esi:include src="..."/>`.
- Sans proxy compatible ESI, `render_esi` retombe sur un rendu **inline** (`InlineFragmentRenderer`) : la page marche, sans gain de cache.
- `ResponseCacheStrategy` combine les fragments : la page finale prend la **plus petite** fraîcheur parmi les parties ; un fragment `private`/`no-store` rend le tout non partagé.
- URL de fragment protégée par une **signature** (`UriSigner`, `_hash`) : sinon n'importe qui pourrait appeler `/_fragment` ; en production, restreindre l'accès aux IP du proxy.
- Variantes : SSI (`framework.ssi: true`, `Ssi.php`), hinclude (JS côté client).

## 7. Sessions et cache

Quand la session est effectivement **utilisée** sur la requête principale, `AbstractSessionListener` rend la réponse privée et ajoute `must-revalidate`. Si elle était publique, son `max-age` devient 0 ; sinon, la durée calculée peut être conservée (sauf désactivation interne via `Symfony-Session-NoAutoCacheControl`). Piège classique : démarrer la session (ou le flash bag, CSRF dans un formulaire) dans une page censée être publique.

## Ce qui change vs Symfony 6/7
- `#[Cache]` (HttpKernel) : depuis 6.2 plus besoin de `sensio/framework-extra-bundle`. **8.1** : paramètre `if`, closures, variables `request` et `args` dans les expressions.
- 8.1 : l'option `framework.http_cache.terminate_on_cache_hit` est **dépréciée** (suppression en 9.0).
- 8.1 : l'affectation directe de `Response::$headers` est dépréciée (passer le `ResponseHeaderBag` au constructeur).
- Le paramétrage `framework.http_cache` sous forme de tableau remplace l'ancien emballage manuel du Kernel (disponible depuis 6.x).

## Piège certification
- `s-maxage` ignore les caches privés ; `max-age` s'applique aux deux. Un cache partagé qui voit `s-maxage` ignore `max-age` et `Expires`.
- `private` => pas de stockage par un cache **partagé**, mais le navigateur peut cacher. `no-store` => personne.
- `no-cache` ne veut **pas** dire « ne pas cacher » : cacher mais revalider.
- 304 : pas de corps ; conserver les métadonnées de validation/cache pertinentes, notamment `ETag`, `Cache-Control` et `Vary` lorsqu'elles sont applicables. Sans `isNotModified()` appelé avant le gros travail, on ne gagne rien côté serveur.
- Seuls certains statuts sont cachables par défaut (`Response::isCacheable()` : 200, 203, 300, 301, 302, 404, 410) et il faut un validateur ou une fraîcheur.
- Une session effectivement utilisée sur la requête principale entraîne la protection automatique `private` ; l'injection de la pile seule ne démarre pas la session.
- Enrichissement hors certification — ESI : sans `framework.esi: true`, `render_esi` n'émet pas de tag ESI ; la route `/_fragment` doit être activée (`fragments`).
- `Vary: Cookie` peut rendre un cache partagé peu efficace et ne remplace jamais une politique d'accès.

## Observer dans l'app

Le profiler n'est **pas installé** dans ce dépôt. Les commandes, traces et tests fonctionnent sans lui ; les observations via `/_profiler` ou `enableProfiler()` ci-dessous sont optionnelles, après installation volontaire avec `composer require --dev symfony/profiler-pack` (Flex). Commencer par `var/log/dev.log` lorsqu'il existe.
- `bin/console debug:config framework http_cache`, `debug:config framework esi`, `debug:config framework fragments`.
- `bin/console debug:router | grep fragment` : route `/_fragment` quand `fragments` activé.
- `curl -I http://localhost:8000/...` (ou `curl -sD - -o /dev/null`) pour lire `Cache-Control`, `ETag`, `Age`, `X-Symfony-Cache`. Rejouer avec `-H 'If-None-Match: "..."'` pour obtenir un 304.
- Profiler (`/_profiler`) : onglet *Request / Response* → en-têtes ; panneau *Cache* pour les pools applicatifs (distinct du cache HTTP).
- Tests : `$client->request('GET', '/x'); $this->assertResponseHeaderSame('Cache-Control', ...)`, puis seconde requête avec `HTTP_IF_NONE_MATCH` et `assertResponseStatusCodeSame(304)`.
- `bin/console debug:event-dispatcher` / `debug:event-dispatcher kernel.response` : retrouver `CacheAttributeListener`.

## Doc officielle
- https://symfony.com/doc/current/http_cache.html
- https://symfony.com/doc/current/http_cache/expiration.html
- https://symfony.com/doc/current/http_cache/validation.html
- https://symfony.com/doc/current/http_cache/esi.html
- https://symfony.com/doc/current/http_cache/cache_vary.html
- https://symfony.com/doc/current/reference/attributes.html (attribut `#[Cache]`)
- https://www.rfc-editor.org/rfc/rfc9111


## 8. Raisonner avec une chronologie et deux utilisateurs

Une réponse stockable n'est pas nécessairement fraîche. Pour `max-age=60, s-maxage=600` et `Age: 45`, le navigateur dispose conceptuellement de 15 secondes de fraîcheur restante, le cache partagé de 555 secondes (en négligeant le délai de transit). `Age` est un âge déjà consommé, pas une prolongation. À expiration, un validateur peut permettre un 304 ; sans validateur exploitable, il faut transférer une nouvelle représentation.

```php
$response = new Response('catalogue');
$response->setPublic()->setMaxAge(60)->setSharedMaxAge(600);
$response->headers->set('Age', '45');
// getTtl() suit ici s-maxage : 555 ; getMaxAge() vaut 600.
```

`Response::getTtl()` ne simule pas tous les types de caches : son calcul privilégie `s-maxage`. Vérifier séparément les règles du navigateur et du proxy. Pour un test déterministe, fixer `Date` et `Age` ; `MockClock` n'intercepte pas magiquement tous les appels à l'horloge système dans `Response`.

### Matrice de revalidation

| Requête conditionnelle, réponse avec ETag et Last-Modified | Résultat attendu |
|---|---|
| ETag identique | 304, corps vide |
| ETag différent et date pourtant inchangée | 200 : la comparaison ETag est prioritaire |
| Date seule égale à Last-Modified | 304 |
| Version serveur changée | 200, nouvelle représentation et nouveau validateur |

Un ETag doit varier avec toute donnée qui change la représentation, notamment langue et format. Un ETag fondé uniquement sur un identifiant immuable est faux lorsque le contenu change. `Vary` choisit la variante stockée ; l'ETag valide cette variante. L'un ne remplace pas l'autre.

### Cookies, sessions et authentification

Une page de catalogue réellement identique pour tous peut être publique. Une page de compte affichant un nom ou un panier est personnelle : préférer `private` et, si aucun stockage n'est souhaité, `no-store`. `Vary: Cookie` multiplie les variantes et n'est pas un contrôle d'autorisation. Un `public` ajouté sans réfléchir peut rendre partageable une réponse à une requête `Authorization`/`Cookie` : les garde-fous du proxy ne dispensent pas de définir la politique métier.

Le listener de session applique sa protection aux requêtes principales selon l'utilisation effective de la session ; injecter `RequestStack` seul n'a pas cet effet. Ne pas utiliser l'en-tête interne `Symfony-Session-NoAutoCacheControl` pour faire taire une protection sans examiner le contenu. `Set-Cookie` ne signifie pas, dans la norme HTTP, « toujours non stockable » : le proxy Symfony ignore par défaut cet en-tête à la restitution depuis son store. Pour des réponses sensibles, imposer explicitement la politique et tester deux utilisateurs distincts.

### Une preuve à chaque niveau

1. Test du contrôleur : en-têtes, version, 304 et absence de rendu coûteux.
2. Test de service : calcul de politique avec données contrôlées.
3. Test du proxy : deuxième appel servi par le store, compteur d'origine inchangé, variantes isolées.

Deux appels de `KernelBrowser` au Kernel applicatif ne prouvent pas un hit du reverse proxy. Le montage de test doit effectivement faire passer les deux requêtes par **la même instance de `HttpCache` et son store isolé** ; consulter `FrameworkBundle/KernelBrowser.php`, le Runtime et `HttpCache::handle()`. L'option `framework.http_cache` seule n'est pas une preuve de ce montage dans un test. Garder le test de proxy distinct des tests fonctionnels de contrôleur.

Sources : [cache HTTP Symfony 8.0](https://symfony.com/doc/8.0/http_cache.html), `vendor/symfony/http-foundation/Response.php`, `vendor/symfony/http-kernel/HttpCache/HttpCache.php` et `EventListener/AbstractSessionListener.php`.
