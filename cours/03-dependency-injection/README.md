# Module 03 — DependencyInjection avancée

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

> **Périmètre** : application vérifiée en **Symfony 8.1.8**, PHP ≥ 8.4 ; la [certification Symfony 8](https://certification.symfony.com/exams/symfony.html) vise **Symfony 8.0**. Les nouveautés marquées **8.1** sont un complément de pratique, pas des exigences de cet examen. Suivre la [méthode de travail du parcours](../README.md) : prédire, observer, tester, expliquer.

## Objectifs et prérequis

Prérequis : interfaces PHP et [cycle du Kernel](../02-kernel-architecture/README.md). Savoir choisir une dépendance sans ambiguïté, lire un graphe de services compilé et prévoir l'ordre des décorateurs. Le scénario de notification est simulé : aucun email, SMS ou message externe réel n'est envoyé.

Hypothèse de base : tu connais l'autowiring et `services.yaml`. Ici : ce qui se passe à la **compilation** du container et les outils avancés (tags, locators, décoration, factories, lazy). Vérifié sur Symfony 8.1 (`vendor/symfony/dependency-injection`).

## 1. Cycle de vie du container

1. Chargement des bundles/extensions et de la configuration → `ContainerBuilder`.
2. **Compilation** : suite de *compiler passes* regroupées par phase (`PassConfig`) :
   - `TYPE_BEFORE_OPTIMIZATION` (défaut quand on ajoute une passe), `TYPE_OPTIMIZE`, `TYPE_BEFORE_REMOVING`, `TYPE_REMOVE`, `TYPE_AFTER_REMOVING`.
   - Entre autres : résolution des définitions enfants, `AutowirePass`, `DecoratorServicePass`, `ResolveTaggedIteratorArgumentPass`, `InlineServiceDefinitionsPass`, `RemoveUnusedDefinitionsPass`.
3. Le container compilé est **dumpé en PHP** dans `var/cache/<env>/` : à l'exécution, plus de réflexion ni de config.

Conséquences : les erreurs de câblage sont levées à la compilation ; les services privés **non utilisés** sont supprimés ; les services privés peuvent être inlinés.

```php
// src/Kernel.php — enregistrer une compiler pass
protected function build(ContainerBuilder $container): void
{
    $container->addCompilerPass(new CollectHandlersPass());      // phase par défaut
    $container->addCompilerPass(new LatePass(), PassConfig::TYPE_BEFORE_REMOVING, priority: 10);
}
```
(`Kernel::build()` est `protected` dans `AbstractKernel` ; dans un bundle on surcharge `AbstractBundle::build()`. Nouveauté 8.1 : support des bundles comme compiler pass.)

```php
final class CollectHandlersPass implements CompilerPassInterface
{
    public function process(ContainerBuilder $container): void
    {
        if (!$container->has(HandlerRegistry::class)) {
            return;
        }
        $registry = $container->findDefinition(HandlerRegistry::class);
        foreach ($container->findTaggedServiceIds('app.handler') as $id => $tags) {
            $registry->addMethodCall('add', [new Reference($id)]);
        }
    }
}
```

## 2. Tags et autoconfiguration

Un **tag** = un marqueur sur une définition, avec des attributs, lu par une passe (ou par un `tagged_iterator`).

```php
#[AutoconfigureTag('app.handler', ['priority' => 10])]   // sur l'interface ou la classe
interface Handler {}

#[AsTaggedItem(index: 'pdf', priority: 20)]               // index et priorité dans l'itérable
final class PdfHandler implements Handler {}
```

- `#[AutoconfigureTag(name, attributes)]` : à poser sur une **interface/classe parente** : toutes les implémentations autoconfigurées reçoivent le tag.
- `#[Autoconfigure(tags:, calls:, bind:, lazy:, public:, shared:, autowire:, properties:, configurator:, constructor:, resourceTags:)]` : même idée, plus large. Répétable.
- `#[Exclude]` sur une classe : ne pas l'enregistrer comme service.
- `#[AsTaggedItem(index:, priority:)]` : priorité (plus haut = plus tôt) et clé. **8.1** : remplace les anciens `defaultIndexMethod` / `defaultPriorityMethod`, **dépréciés**.
- `_instanceof` dans `services.yaml` : équivalent configurable de l'autoconfiguration.
- `registerForAutoconfiguration()` (dans une extension de bundle) pour tes propres tags.

## 3. Tagged iterators et ServiceLocator

```php
use Symfony\Component\DependencyInjection\Attribute\AutowireIterator;
use Symfony\Component\DependencyInjection\Attribute\AutowireLocator;
use Symfony\Component\DependencyInjection\ServiceLocator;

final class Pipeline
{
    public function __construct(
        #[AutowireIterator('app.handler')] private iterable $handlers,           // paresseux, ordre = priorité
        #[AutowireLocator('app.handler', indexAttribute: 'key')] private ServiceLocator $locator,
    ) {}
}
```

- **Iterator** : parcourt les services tagués, chacun n'étant instancié qu'au moment de l'itération. `iterable` ou `\Traversable`.
- **ServiceLocator** (implémente PSR-11 : `get`, `has`) : accès par clé à la demande, services instanciés au `get()` ; `getProvidedServices()` donne la liste id => type.
- Paramètres de `#[AutowireIterator]`/`#[AutowireLocator]` : `tag`/`services`, `indexAttribute`, `exclude`, `excludeSelf` (par défaut `true`).
- `#[AutowireLocator([...])]` avec un tableau explicite : `['pdf' => PdfHandler::class, 'csv' => new Autowire(service: 'app.csv')]`, ou `SubscribedService`.
- YAML équivalent : `!tagged_iterator { tag: app.handler, index_by: key }` et `!tagged_locator`.
- **8.0** : `#[TaggedIterator]` et `#[TaggedLocator]` sont **supprimés** → `#[AutowireIterator]` / `#[AutowireLocator]`. Le `!tagged` YAML est supprimé aussi (utiliser `!tagged_iterator`).

### ServiceSubscriber

```php
final class Mailer implements ServiceSubscriberInterface
{
    public function __construct(private ContainerInterface $locator) {}   // locator injecté automatiquement

    public static function getSubscribedServices(): array
    {
        return [LoggerInterface::class, 'twig' => Environment::class, new SubscribedService('http', HttpClientInterface::class, nullable: true)];
    }
}
```

- Le container injecte un `ServiceLocator` limité aux services listés (tag `container.service_subscriber`).
- `AbstractController` utilise ce mécanisme (via `ServiceMethodsSubscriberTrait`) : c'est pourquoi `$this->container->get()` n'accède qu'à ses services déclarés.
- `ServiceSubscriberTrait` est **déprécié** depuis `symfony/contracts` 3.5 : utiliser `ServiceMethodsSubscriberTrait` (méthodes annotées `#[SubscribedService]`).
- En pratique moderne : `#[AutowireLocator]` est plus simple que `ServiceSubscriberInterface`.

## 4. Décoration

```php
#[AsDecorator(decorates: Mailer::class, priority: 10)]
final class LoggingMailer implements MailerInterface
{
    public function __construct(
        #[AutowireDecorated] private MailerInterface $inner,   // service décoré
        private LoggerInterface $logger,
    ) {}
}
```
- `#[AsDecorator(decorates, priority = 0, onInvalid)]` (répétable depuis 7.4). YAML : `decorates: App\Mailer`, `decoration_inner_name`, `decoration_priority`, `decoration_on_invalid`.
- L'ancien service reçoit l'id `<id du décorateur>.inner` (ou `decoration_inner_name`) ; l'**id d'origine** pointe désormais sur le décorateur (alias).
- Plusieurs décorateurs : **priorité la plus haute appliquée en premier**, donc au plus près du service d'origine (le décorateur de priorité la plus basse est le plus externe). Voir `DecoratorServicePass` (file de priorité : plus haute extraite d'abord).
- `#[AutowireDecorated]` (ou `service('.inner')` / `@.inner` en YAML) injecte l'interne.
- **8.1** : *service stacks* comme décorateurs (`decorates_tag`) ; décoration de **tous les services d'un tag** avec `#[AsTagDecorator(tag, priority, onInvalid)]` ou la resource tag `container.tag_decorator`.
- `debug:container <id>` : en 8.1 affiche la **pile de décoration**.

## 5. Factories, Autowire, Closures

```yaml
services:
    App\Report\Exporter:
        factory: ['@App\Report\ExporterFactory', 'create']   # service factory
        arguments: ['%kernel.environment%']
    App\Clock\Snapshot:
        factory: ['App\Clock\Snapshot', 'now']               # static factory
```
Variante PHP : `constructor: 'create'` dans `#[Autoconfigure(constructor: 'create')]` (méthode statique publique), `->factory([...])`.

Attributs sur les paramètres de constructeur :
- `#[Autowire(service: 'x')]`, `#[Autowire(param: 'kernel.project_dir')]`, `#[Autowire(env: 'APP_SECRET')]`, `#[Autowire(expression: 'service("x").foo()')]`, `#[Autowire(value: '%kernel.project_dir%/var')]`, `lazy: true`. **Un seul** parmi service/expression/env/param/value.
- `#[AutowireServiceClosure('x')]` : un `\Closure` qui retourne le service (instancié à l'appel).
- `#[AutowireCallable(service: 'x', method: 'm')]` : un `\Closure` vers une méthode d'un service. 8.1 : `from_callable` dans la config est restreint (options dépréciées).
- `#[AutowireInline(...)]` : définit un service anonyme à la volée.
- `#[Target('name')]` : choisit un **alias nommé** quand plusieurs services implémentent la même interface. **8.1** : la sélection implicite des alias nommés par le seul nom de paramètre est dépréciée ; `#[AsAlias(target: ...)]` crée des alias ciblés.
- **8.1** : `#[Autowire(env: ...)]` peut aussi injecter un `\Closure`/`Stringable` (variable d'env évaluée à la demande).

## 6. Lazy services

```php
#[Lazy]                                    // sur la classe : service lazy
final class HeavyService {}

public function __construct(#[Lazy] private HeavyService $heavy) {}   // lazy pour ce paramètre
```
- Le service injecté est un proxy/ghost ; la vraie instance est construite lorsque l'accès déclenche son initialisation (notamment la lecture de son état ; une méthode sans accès à l'état peut ne pas l'initialiser).
- S'appuie sur les **lazy objects natifs de PHP 8.4** quand c'est possible (changelog DI : « Leverage native lazy objects when possible ») ; plus de dépendance à `ocramius/proxy-manager` depuis longtemps (module 01).
- `lazy: true` (YAML/PHP) ou `lazy: 'Interface'` pour proxifier une interface (les objets paresseux natifs de PHP 8.4 permettent aussi les classes utilisateur `final` ; une interface n'est donc pas systématiquement nécessaire).
- `#[Lazy]` = raccourci de `#[Autowire(lazy: ...)]` / `#[Autoconfigure(lazy: ...)]`.
- Intérêt : éviter d'instancier un service coûteux (connexion, client) quand il n'est pas toujours utilisé ; alternatives : `ServiceLocator`, `AutowireServiceClosure`.

## 7. Environnements : `#[When]` / `#[WhenNot]`

```php
#[When(env: 'dev')]
#[When(env: 'test')]
final class FakeSmsGateway implements SmsGateway {}   // enregistré seulement en dev/test

#[WhenNot(env: 'prod')]
```
- Répétables, sur classe/méthode/fonction. Fonctionne avec `resource:` dans `services.yaml` (le fichier est analysé).
- `#[AsAlias(id: 'x', when: ['dev', 'test'])]` : alias par environnement.
- YAML : `when@test:` / `when@dev:` pour des blocs entiers.

## 8. Autres notions à connaître

- **Services publics/privés** : tout est privé par défaut ; `$container->get()` d'un privé échoue en prod, mais `static::getContainer()` en test donne un container qui expose les privés **utilisés**. Services supprimés si inutilisés : `static::getContainer()->has(...)` peut être faux.
- `_defaults` (`autowire`, `autoconfigure`, `public`, `bind`) ; `bind` par nom/type de paramètre (`$projectDir: '%kernel.project_dir%'`).
- Alias : `App\Foo\Interface: '@App\Foo\Impl'` ou `#[AsAlias]`. `ResolveReferencesToAliasesPass`.
- `kernel.reset` : services avec `ResetInterface` remis à zéro entre requêtes (8.1 : peut aussi s'appliquer aux services non partagés).
- Détection de cycle : `CheckCircularReferencesPass`. Revoir d'abord les responsabilités ; setter/lazy/locator peuvent différer une arête, mais ne réparent pas une récursion métier.
- `shared: false` : une nouvelle instance à chaque injection/`get`.
- `#[Required]` (Contracts) sur setter/propriété : injection de setter autowirée.

## Ce qui change vs Symfony 6/7

- `#[TaggedIterator]` / `#[TaggedLocator]` supprimés en 8.0 → `#[AutowireIterator]` / `#[AutowireLocator]`.
- Format **XML** de configuration supprimé en 8.0 (YAML ou PHP) ; `$this` interdit dans les fichiers PHP de config (variable `$loader`).
- `ContainerBuilder::getAutoconfiguredAttributes()` supprimé → `getAttributeAutoconfigurators()`.
- 8.1 : `defaultIndexMethod`/`defaultPriorityMethod` dépréciés au profit de `#[AsTaggedItem]` ; `#[AsTagDecorator]` ; stacks décorateurs ; sélection implicite par nom de paramètre dépréciée au profit de `#[Target]` ; `ResettableServicePass`/`ServicesResetter` migrés dans DI ; `Kernel`/`Bundle` de base dans `DependencyInjection\Kernel`.
- `#[AsDecorator]` répétable depuis 7.4 ; `#[Lazy]` existe depuis 7.1.

## Piège certification
- **Priorité de tag** : `priority` plus élevée = plus tôt dans l'itérateur. **Décoration** : priorité plus élevée = appliqué en premier (plus interne).
- `#[AutowireIterator]` ≠ `array` : c'est un `iterable`/`Traversable` (paresseux et réitérable avec le `RewindableGenerator` généré), pas un tableau ; utiliser `iterator_to_array()`.
- Un service **non utilisé et privé** est supprimé du container compilé : `debug:container` peut encore montrer sa définition de diagnostic et signaler sa suppression ou son inlining ; cela ne prouve pas qu'un `get()` soit possible.
- `ServiceLocator` n'expose que `get`/`has` (PSR-11), pas `set`.
- `#[Autoconfigure]` / `#[AutoconfigureTag]` doivent être sur la **classe ou l'interface** parente, pas sur le consommateur.
- `excludeSelf` vaut `true` par défaut : un service tagué ne se reçoit pas lui-même.
- Passer une compiler pass dans `build()` : le type de passe (phase) change quand elle s'exécute par rapport à l'autowiring/décoration.

## Observer dans l'app

Le profiler n'est **pas installé** dans ce dépôt. Les commandes, traces et tests fonctionnent sans lui ; les observations via `/_profiler` ou `enableProfiler()` ci-dessous sont optionnelles, après installation volontaire avec `composer require --dev symfony/profiler-pack` (Flex). Commencer par `var/log/dev.log` lorsqu'il existe.
- `bin/console debug:container` ; `debug:container <id>`.
- `bin/console debug:container --tag=app.handler` ; `--tags` (liste de tous les tags) ; `--types`.
- `bin/console debug:autowiring <mot>` : alias et interfaces autowirables.
- `bin/console lint:container` : vérifie les types des arguments injectés.
- `bin/console debug:container --env-vars`, `--parameters`, `--deprecations`.
- Fichiers générés : `var/cache/dev/App_KernelDevDebugContainer.xml` (graphe de définitions) et les classes `Container*/` (vue réelle des factories, inlining, lazy).
- Tests : `KernelTestCase` + `static::getContainer()->get(Id::class)`, `$container->has()`, assert sur la liste d'un locator (`getProvidedServices()`).
- `ContainerBuilder` pur en test unitaire d'une compiler pass : `new ContainerBuilder()` + `register()` + `addCompilerPass()` + `compile()`.

## Doc officielle
- https://symfony.com/doc/current/service_container.html
- https://symfony.com/doc/current/service_container/compiler_passes.html
- https://symfony.com/doc/current/service_container/tags.html
- https://symfony.com/doc/current/service_container/service_subscribers_locators.html
- https://symfony.com/doc/current/service_container/service_decoration.html
- https://symfony.com/doc/current/service_container/factories.html
- https://symfony.com/doc/current/service_container/lazy_services.html
- https://symfony.com/doc/current/service_container/autowiring.html
- https://symfony.com/doc/current/configuration.html (`when@env`, environnements)


## 9. Du type PHP au graphe de notification

L'autowiring cherche quel service correspond à un type ; l'autoconfiguration ajoute des métadonnées à partir des interfaces ou attributs. Un tag de canal ne choisit pas automatiquement le canal par défaut de `Notifier`. Avec `EmailChannel` et `SmsChannel` pour `ChannelInterface`, un argument `ChannelInterface $channel` demande un choix explicite.

```php
// Extrait de constructeur d'un consommateur : choix local, compatible 8.0.
public function __construct(
    #[Autowire(service: SmsChannel::class)] private ChannelInterface $channel,
) {}
```

Un alias de l'interface définit le choix par défaut pour tous les consommateurs ; `#[Target]` sélectionne un alias nommé ; un locator répond au besoin « choisir un canal au moment de l'appel ». Un itérateur répond à « appeler tous les canaux ». Ce sont des contrats métier différents, pas des syntaxes interchangeables.

### Ordre des décorateurs et ordre des appels

Si `RetryChannel` a priorité 20 et `LoggingChannel` priorité 0, le graphe est `Logging(Retry(Sms))`. Une trace avant/après devient `log.before`, `retry.before`, `sms`, `retry.after`, `log.after`. Si l'interne lève une exception, les appels après ne surviennent que si le code les garantit (`finally`, par exemple). Tester une réussite seule ne prouve donc pas le comportement d'un décorateur en échec.

Le nom interne par défaut dépend du **décorateur**, par exemple `App\Di\LoggingChannel.inner`. Utiliser `#[AutowireDecorated]` évite de l'écrire dans le constructeur. En **8.1**, `debug:container` facilite la lecture de la pile de décoration ; l'ordre métier doit aussi être prouvé par un test de service.

### Privé, supprimé et introuvable ne sont pas synonymes

| Observation | Explication à rechercher |
|---|---|
| Type refusé lors de la compilation | interface ambiguë, service exclu ou argument scalaire non configuré |
| Définition visible au diagnostic, `get()` impossible | service privé, supprimé ou intégré dans un consommateur (*inlining*) |
| Service absent uniquement dans un environnement | `#[When]`, `#[WhenNot]`, alias/configuration de cet environnement |
| `A → B → A` | cycle de construction ou responsabilités trop couplées |

Le container de test donne accès aux services privés encore disponibles ; il ne recrée pas une définition supprimée. Pour tester un canal, passer d'abord par le `Notifier` qui le consomme. Un alias public **uniquement en test** peut se justifier pour un besoin d'accès précis ; rendre tous les services publics masque le diagnostic.

Sur un cycle `Notifier → Channel → Notifier`, dessiner la responsabilité de chacun. Extraire un petit service commun peut supprimer le cycle. Un locator ou un lazy ne doit pas être invoqué immédiatement depuis le constructeur pour contourner la compilation : le cycle reviendrait à l'exécution.

Parcours d'observation : `debug:autowiring Channel`, `debug:container 'App\Di\Notifier'`, `debug:container --tag=app.notification_channel`, puis `lint:container --env=test`. Utiliser des noms de classe avec un seul antislash en shell entre apostrophes. Ne jamais modifier les classes générées dans `var/cache/` : corriger les attributs ou les définitions sources. Voir les [bonnes pratiques Symfony 8.0](https://symfony.com/doc/8.0/best_practices.html).
