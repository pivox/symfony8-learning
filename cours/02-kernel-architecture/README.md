# Module 02 — Kernel & architecture

> **Périmètre** : application vérifiée en **Symfony 8.1.8**, PHP ≥ 8.4 ; la [certification Symfony 8](https://certification.symfony.com/exams/symfony.html) vise **Symfony 8.0**. Les nouveautés marquées **8.1** sont un complément de pratique, pas des exigences de cet examen. Suivre la [méthode de travail du parcours](../README.md) : prédire, observer, tester, expliquer.

## Objectifs et prérequis

Prérequis : [PHP et interfaces](../01-php84-psr/README.md), `Request`, `Response` et attribut `Route`. Savoir reconstituer une trace d'événements, expliquer une réponse anticipée et une exception, puis prouver que les sous-requêtes ne laissent pas d'état dans `RequestStack`.

Comment une requête HTTP devient une réponse : du point d'entrée `public/index.php` jusqu'aux événements `kernel.*`. Versions vérifiées dans ce dépôt : Symfony **8.1** (`Kernel::VERSION`).

## 1. Point d'entrée et Runtime

```php
// public/index.php
require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return static function (array $context) {
    return new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
};
```

- `autoload_runtime.php` est généré par `symfony/runtime` : le script **retourne un callable** (ni `new Kernel()->handle()`, ni `->send()` écrits à la main).
- Le Runtime résout les arguments du callable (`array $context`, `Request`, etc.), puis choisit un *Runner* selon le type de l'objet retourné. Pour un `HttpKernelInterface`, c'est `HttpKernelRunner` : `handle()` → `send()` → `fastcgi_finish_request()` (si disponible et hors debug) → `terminate()`.
- `$context` contient `APP_ENV`, `APP_DEBUG` (issus de `$_SERVER`/`$_ENV` et du `.env` via Dotenv).
- Avantage : le même code tourne sous FPM, FrankenPHP, Swoole, CLI, etc., en changeant seulement le runtime.

## 2. Le Kernel de l'application

```php
class Kernel extends BaseKernel
{
    use MicroKernelTrait;
}
```

- `Symfony\Component\HttpKernel\Kernel` : gère bundles, container, cache (`kernel.cache_dir`, `kernel.build_dir`, `kernel.share_dir`), `boot()`, `shutdown()`, `handle()` (délègue à `HttpKernel` récupéré dans le container) et `terminate()`.
- `MicroKernelTrait` (FrameworkBundle) : charge `config/packages/*.yaml`, `config/services.yaml`, `config/routes.yaml`, `config/bundles.php` ; fournit `registerBundles()`, `getLogDir()`, `loadRoutes()`.
- `Kernel::build(ContainerBuilder $container)` (protected) est le point d'extension pour enregistrer des **compiler passes** directement dans le Kernel (module 03).
- `getAllowedEnvs()` (nouveau 8.1 dans `MicroKernelTrait`, cf. `src/Kernel.php`) liste les valeurs acceptées pour `APP_ENV`.

## 3. `HttpKernel::handle()` — le cycle de vie

`HttpKernelInterface::handle(Request $request, int $type = MAIN_REQUEST, bool $catch = true): Response`

Séquence réelle (lue dans `HttpKernel::handleRaw()`) :

```
Request
  │  kernel.request            (RequestEvent)               ← un listener peut poser une Response => court-circuit (saute controller)
  │  ControllerResolver::getController()                    ← false => NotFoundHttpException
  │  kernel.controller         (ControllerEvent)            ← setController() possible
  │  ArgumentResolver::getArguments()
  │  kernel.controller_arguments (ControllerArgumentsEvent) ← setArguments() possible
  │  $controller(...$arguments)
  │  [si le retour n'est pas une Response]
  │  kernel.view               (ViewEvent)                  ← doit poser une Response, sinon ControllerDoesNotReturnResponseException
  │  kernel.response           (ResponseEvent)
  │  kernel.finish_request     (FinishRequestEvent)
Response → send() → kernel.terminate (TerminateEvent)  [après envoi]
  (en cas de Throwable)  kernel.exception (ExceptionEvent) → puis response + finish_request
```

| Constante `KernelEvents::` | Nom | Classe d'événement | Usage typique |
|---|---|---|---|
| `REQUEST` | `kernel.request` | `RequestEvent` | routing, locale, firewall, retour anticipé d'une réponse |
| `CONTROLLER` | `kernel.controller` | `ControllerEvent` | remplacer/envelopper le controller |
| `CONTROLLER_ARGUMENTS` | `kernel.controller_arguments` | `ControllerArgumentsEvent` | modifier les arguments résolus |
| `VIEW` | `kernel.view` | `ViewEvent` | convertir un retour non-`Response` (array, DTO) en `Response` |
| `RESPONSE` | `kernel.response` | `ResponseEvent` | modifier headers, cookies, profiler |
| `FINISH_REQUEST` | `kernel.finish_request` | `FinishRequestEvent` | nettoyer l'état (locale, request stack) |
| `TERMINATE` | `kernel.terminate` | `TerminateEvent` | travail de fin de requête (synchrone, sans garantie de livraison) |
| `EXCEPTION` | `kernel.exception` | `ExceptionEvent` | convertir une exception en `Response` |

Points à retenir :
- `kernel.view` n'est dispatché **que** si le controller ne retourne pas une `Response`.
- `kernel.finish_request` est dispatché **dans** `filterResponse()`, donc après `kernel.response`, et aussi pour les sous-requêtes.
- `kernel.terminate` n'est pas dans `handle()` : c'est `terminate()`, appelé par le Runner **après** l'envoi (`fastcgi_finish_request` hors debug). `ExceptionEvent::isKernelTerminating()` indique si l'exception survient pendant le terminate.
- `handle($request, $type, catch: false)` : les exceptions ne sont pas converties en réponse (pour une exception ordinaire, `kernel.finish_request` est dispatché, puis l'exception est relancée). Les `\Error` PHP sont relancées telles quelles (sauf option `handleAllThrowables`).
- `RequestExceptionInterface` (ex. paramètre malformé) est converti en `BadRequestHttpException` (400).

### Priorités par défaut de quelques listeners (lues dans le code)

Plus la priorité est **élevée**, plus le listener s'exécute **tôt** (défaut : `0`).

| Événement | Listener | Priorité |
|---|---|---|
| `kernel.request` | `DebugHandlersListener::configure` | 2048 |
| `kernel.request` | `ValidateRequestListener` | 256 |
| `kernel.request` | `AddRequestFormatsListener` | 100 |
| `kernel.request` | `LocaleListener::setDefaultLocale` | 100 |
| `kernel.request` | `AbstractSessionListener` | 128 |
| `kernel.request` | `FragmentListener` | 48 |
| `kernel.request` | `RouterListener` | 32 |
| `kernel.request` | `LocaleListener::onKernelRequest` | 16 (après le Router pour lire `_locale`) |
| `kernel.controller_arguments` | `ControllerAttributesListener` | -10000 |
| `kernel.controller_arguments` | `RequestPayloadValueResolver` | -10100 |
| `kernel.exception` | `ErrorListener::logKernelException` | 0 |
| `kernel.exception` | `RouterListener` (page d'accueil en debug si aucune route configurée) | -64 |
| `kernel.exception` | `ErrorListener::onKernelException` | -128 |
| `kernel.response` | `ProfilerListener` | -1024 |
| `kernel.response` | `AbstractSessionListener` | -1000 |

(La liste est partielle : le firewall de sécurité et d'autres bundles s'ajoutent.) Vérifie toujours avec `debug:event-dispatcher`.

### Écrire un listener

```php
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\KernelEvents;

final class PoweredByListener
{
    #[AsEventListener(event: KernelEvents::RESPONSE, priority: -10)]
    public function __invoke(ResponseEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return;                              // ignorer les sous-requêtes
        }
        $event->getResponse()->headers->set('X-Powered-By', 'Symfony 8');
    }
}
```

Alternative : `EventSubscriberInterface::getSubscribedEvents()` (`['kernel.response' => ['onResponse', -10]]`). Avec l'autoconfiguration (`config/services.yaml`), le tag `kernel.event_listener` / `kernel.event_subscriber` est posé automatiquement.

`#[AsEventListener]` : paramètres `event`, `method`, `priority` (défaut 0), `dispatcher`.

## 4. Controller et ArgumentResolver

`ControllerResolver` lit l'attribut de requête `_controller` (posé par `RouterListener`) et en fait un callable. Puis `ArgumentResolver` appelle chaque `ValueResolverInterface` (méthode `resolve(Request, ArgumentMetadata): iterable`) pour chaque paramètre ; le premier qui *yield* une valeur gagne (au plus une valeur pour un argument non variadique). Si aucun resolver ne résout un argument : `RuntimeException` « Controller ... requires the "$x" argument that could not be resolved ».

Resolvers livrés par FrameworkBundle (tag `controller.argument_value_resolver`, priorités lues dans `web.php`) :

| Resolver | Priorité |
|---|---|
| `RequestValueResolver` (type `Request`) | 120 |
| `SessionValueResolver` | 120 |
| `BackedEnumValueResolver`, `UidValueResolver`, `DateTimeValueResolver`, `RequestAttributeValueResolver` | 100 |
| `ServiceValueResolver` (services via `controller.service_arguments`) | -50 |
| `DefaultValueResolver` (valeur par défaut du paramètre) | -100 |
| `VariadicValueResolver` | -150 |

Les resolvers `#[MapQueryParameter]`, `#[MapQueryString]`, `#[MapRequestPayload]`, `#[MapRequestHeader]`, `#[MapUploadedFile]`, `#[MapDateTime]` sont déclenchés explicitement par un attribut qui étend `ValueResolver` (module 04 pour le détail HTTP).

Resolver personnalisé :

```php
final class CurrentTenantResolver implements ValueResolverInterface
{
    public function resolve(Request $request, ArgumentMetadata $argument): iterable
    {
        if ($argument->getType() !== Tenant::class) {
            return [];                      // laisse les autres resolvers essayer
        }
        return [new Tenant($request->attributes->get('tenant'))];
    }
}
```

Avec l'autoconfiguration, une classe implémentant `ValueResolverInterface` reçoit le tag `controller.argument_value_resolver`. Pour qu'un resolver ne s'applique qu'à la demande : attribut `#[AsTargetedValueResolver('nom')]` sur la classe + `#[ValueResolver('nom')]` sur le paramètre (`ValueResolver` a aussi `disabled: true` pour désactiver un resolver global sur un paramètre ; on ne peut « épingler » qu'un seul resolver par argument, sinon `LogicException`).

Un controller est un service si possible : `#[AsController]` (tag `controller.service_arguments`) rend ses méthodes autowirées par argument.

## 5. Sous-requêtes et RequestStack

```php
$subResponse = $kernel->handle($subRequest, HttpKernelInterface::SUB_REQUEST);
```

- `HttpKernelInterface::MAIN_REQUEST = 1`, `SUB_REQUEST = 2`.
- Les événements du traitement `handle()` sont dispatchés aussi pour une sous-requête (`terminate` reste un appel séparé) : `$event->isMainRequest()` permet de filtrer.
- `RequestStack` : `getCurrentRequest()`, `getMainRequest()`, `getParentRequest()`. `handle()` fait `push()` au début et `pop()` dans `finally`.
- Usages : `AbstractController::forward()`, `{{ render(controller('App\\Controller\\Foo::bar')) }}` (Twig), fragments ESI/hinclude.
- Ne **jamais** injecter `Request` dans un service : utilise `RequestStack`.

## 6. Nouveautés 8.x autour du cycle de vie

Constatées dans `vendor/symfony/http-kernel/CHANGELOG.md` (8.1) :
- **Événements nommés d'après les attributs de controller** : `ControllerAttributesListener` dispatche `ControllerAttributeEvent` avec un nom du type `kernel.controller_arguments.<FQCN attribut>` (voir `CacheAttributeListener`, `IsSignatureValidAttributeListener`, `RateLimitAttributeListener`). `ControllerEvent::getAttributes('*')` retourne une liste plate.
- **`#[Serialize]`** : sur un controller, le retour (objet/array) est sérialisé via l'événement `kernel.view` (`SerializeControllerResultAttributeListener`) ; paramètres `code`, `headers`, `context`.
- **`#[RateLimit]`** sur un controller (nécessite `framework.rate_limiter`).
- `#[Cache]` accepte `Closure`/condition ; `#[MapRequestHeader]` ; `$controllerMetadata` sur plusieurs événements ; les paramètres de route typés sont validés avant d'appeler le controller.
- `Kernel`, `Bundle` : l'infrastructure de base migre vers `Symfony\Component\DependencyInjection\Kernel\` (`AbstractKernel`, `AbstractBundle`) ; `HttpKernel\Bundle\BundleInterface`, `FileLocator`, `MergeExtensionConfigurationPass`, `ServicesResetter` sont dépréciés dans HttpKernel au profit des versions DI.
- Les `Bundle::registerCommands()` sont dépréciés : `#[AsCommand]`.

## Ce qui change vs Symfony 6/7

- Les `ArgumentValueResolverInterface` (ancienne API) ne sont plus : seule `ValueResolverInterface` existe.
- `Kernel::getAnnotatedClassesToCompile()`, `AddAnnotatedClassesToCachePass` supprimés (8.0). `KernelInterface::getShareDir()` ajouté.
- Les annotations Doctrine/Route ne sont plus supportées : attributs PHP uniquement.
- `ErrorListener::logException()` reçoit `$logChannel` (8.0).
- Le XML n'est plus un format de configuration (8.0) : YAML ou PHP.
- Dans HttpFoundation 8.0 : `Request::get()` supprimé (utiliser `->attributes`, `->query`, `->request`).

## Piège certification
- Ordre : `request` → `controller` → `controller_arguments` → (controller) → `view` (si pas de `Response`) → `response` → `finish_request` ; `terminate` vient **après** l'envoi, pas dans `handle()`.
- Une priorité plus **haute** s'exécute **avant**. Défaut `0`. Les priorités négatives passent après les priorités supérieures ; des listeners du framework ont eux aussi des priorités négatives.
- `RequestEvent::setResponse()` appelle `stopPropagation()` ; `ViewEvent` et `ExceptionEvent` étendent `RequestEvent` et héritent de ce comportement. `ResponseEvent::setResponse()` remplace la réponse sans arrêter automatiquement la propagation. `TerminateEvent` et `FinishRequestEvent` ne proposent pas ce remplacement (mais `stopPropagation()` reste disponible, car tout événement étend `Event`).
- Dans un listener `kernel.exception`, `$event->setResponse()` remplace le comportement par défaut ; sans réponse, l'exception est relancée.
- `kernel.view` : si personne ne pose de réponse, erreur `ControllerDoesNotReturnResponseException` (et le message suggère « Did you forget to add a return statement ? »).
- Filtrer par `isMainRequest()` dans les listeners qui ne doivent agir qu'une fois.
- Les arguments de controller : l'ordre des resolvers dépend de la **priorité** du tag ; un resolver retournant `[]` passe la main.

## Observer dans l'app

Le profiler n'est **pas installé** dans ce dépôt. Les commandes, traces et tests fonctionnent sans lui ; les observations via `/_profiler` ou `enableProfiler()` ci-dessous sont optionnelles, après installation volontaire avec `composer require --dev symfony/profiler-pack` (Flex). Commencer par `var/log/dev.log` lorsqu'il existe.
- `bin/console about` : version, env, debug, kernel.
- `bin/console debug:event-dispatcher kernel.request` (puis `kernel.controller_arguments`, `kernel.exception`, `kernel.response`) : listeners **dans l'ordre d'exécution** avec priorités.
- `bin/console debug:container --tag=controller.argument_value_resolver` ; `debug:container --tag=kernel.event_listener`.
- `bin/console debug:router` / `debug:router <nom>` : voir le `_controller` associé.
- `bin/console debug:container request_stack` / `debug:autowiring RequestStack`.
- Profiler (`/_profiler`, onglet Events / Request / Performance) : les événements appelés, les listeners non appelés, la timeline. Onglet *Request* : attributs de requête (`_route`, `_controller`).
- Tests : `KernelTestCase::bootKernel()` + `static::getContainer()->get('event_dispatcher')->getListeners('kernel.request')` ; `WebTestCase` + `$client->getProfile()` (après `$client->enableProfiler()`) ; `$kernel->handle(Request::create('/'))` directement.
- `php -r 'require "vendor/autoload.php"; echo Symfony\Component\HttpKernel\Kernel::VERSION;'`.

## Doc officielle
- https://symfony.com/doc/current/components/http_kernel.html
- https://symfony.com/doc/current/reference/events.html
- https://symfony.com/doc/current/event_dispatcher.html
- https://symfony.com/doc/current/controller/value_resolver.html
- https://symfony.com/doc/current/configuration/front_controllers_and_kernel.html
- https://symfony.com/doc/current/components/runtime.html
- https://symfony.com/doc/current/controller/forwarding.html (sous-requêtes via `forward`)


## 7. Lire une trace plutôt que réciter une liste

Les événements sont des points de décision. Pour observer un cas, enregistrer dans un collecteur de test le nom de l'événement, `isMainRequest()` et le chemin de `getRequest()`. Ne pas utiliser une sortie `echo` : elle modifierait la réponse à mesurer. Commencer par une route qui retourne explicitement `new Response('ok')`.

| Scénario contrôlé | Trace attendue, sans les listeners internes |
|---|---|
| Réponse normale | request → controller → controller_arguments → response → finish_request |
| Réponse posée dans request | request → response → finish_request ; aucun contrôleur |
| Exception du contrôleur transformée par ton listener | request → controller → controller_arguments → exception → response → finish_request |
| Exception ordinaire avec `catch: false` | request → controller → controller_arguments → finish_request, puis exception au test |

La trace complète peut contenir une sous-requête de rendu d'erreur créée par `ErrorListener`. Pour tester l'ordre ci-dessus sans cette variation, ton listener traite uniquement une exception métier connue. Dans un test HTTP de statut d'erreur, laisser le client capturer les exceptions ; dans un test d'exception PHP, désactiver cette capture. Ne pas confondre le résultat de ces deux expériences.

### La pile pendant et après une sous-requête

Pendant `/parent` → `/fragment`, `getMainRequest()` reste `/parent`, `getCurrentRequest()` devient `/fragment` et `getParentRequest()` vaut `/parent`. Au retour du fragment, `getCurrentRequest()` redevient `/parent`. Au retour complet de `handle()` appelé sur une pile initialement vide, les trois accès retournent `null`.

Le `pop()` est dans le `finally` de `HttpKernel::handle()`, **après** `kernel.finish_request`. Un listener de fin voit donc encore la requête en cours et peut restaurer l'état du parent. Une exception n'autorise pas à laisser la requête empilée. Attention à la nuance du code installé : un `Error` relancé avec `handleAllThrowables: false` passe par `finally`, mais pas nécessairement par `finishRequest()` ; ne pas confondre garantie de dépilement et garantie d'émission d'un événement.

```php
// Observation après un appel direct au Kernel, hors callback de streaming.
$request = Request::create('/route-existante');
$response = $kernel->handle($request);
assert($requestStack->getCurrentRequest() === null);
```

Récupérer le `RequestStack` du même container que le Kernel avant l'appel. Éviter de conserver la `Request` dans le constructeur d'un service partagé : elle peut être absente en CLI et différente lors de l'appel suivant.

### Terminer n'est pas mettre en file d'attente

`kernel.terminate` reste du travail dans le processus PHP. Le client peut attendre selon le serveur et le mode debug ; un arrêt du processus peut perdre le travail. Pour une tâche qui demande reprise, tentatives et suivi, consulter [Messenger](../08-messenger/README.md) après vérification du parcours. Observer séparément `handle()` puis `terminate()` dans un test direct ; `KernelBrowser` possède son propre cycle et appelle la terminaison.

Sources locales : `vendor/symfony/http-kernel/HttpKernel.php`, `Event/ResponseEvent.php`, `EventListener/ErrorListener.php`, `vendor/symfony/runtime/Runner/Symfony/HttpKernelRunner.php`.
