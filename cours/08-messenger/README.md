# Module 08 — Messenger

[Parcours](../README.md) · [Exercices](exercices.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

## Objectifs, prérequis et repères

> Application : **Symfony 8.1.8, PHP ≥ 8.4**. La [certification Symfony](https://certification.symfony.com/exams/symfony.html) cible **8.0** : les mentions **8.1 uniquement** sont des compléments pour cette application, pas des acquis à attribuer à 8.0.

À la fin du module, tu dois distinguer dispatch synchrone, envoi et consommation, observer un retry complet, expliquer les doublons et empêcher un état de service de contaminer le message suivant.

Prérequis : injection, événements, DTO et exceptions ; [Console](../13-console/README.md) pour le processus worker. Messenger est installé ; aucun broker ni bridge Doctrine n'est requis pour le parcours de base. Le transport `sync://` configuré ne crée pas de file durable.

Applique la [méthode de travail commune](../README.md) : prédire, exécuter, observer, expliquer. Les exemples ci-dessous illustrent un mécanisme ; les exercices restent à réaliser. Le profiler n'est pas installé : utiliser les commandes et les tests proposés. Son installation facultative passerait par `composer require --dev symfony/profiler-pack`.

Messenger = un **bus de messages** : tu `dispatch()` un objet (le message), des **middleware** l'enveloppent, puis un **handler** le traite, soit tout de suite (sync), soit plus tard par un **worker** via un **transport**.

> Le package `symfony/messenger` est déjà installé (8.1). Les transports réels demandent un package en plus : `composer require symfony/doctrine-messenger` (ou `symfony/redis-messenger`, `symfony/amqp-messenger`). Dans cette app, seul le transport `sync://` est configuré (`config/packages/messenger.yaml`) ; `in-memory://` est natif et sert aux tests.

## 1. Message, handler, dispatch

```php
final readonly class SendWelcomeEmail          // un message = un simple objet (DTO), sérialisable
{
    public function __construct(public int $userId) {}
}

#[AsMessageHandler]
final class SendWelcomeEmailHandler
{
    public function __invoke(SendWelcomeEmail $message): void { /* ... */ }
}

// Dans un service / contrôleur
$envelope = $bus->dispatch(new SendWelcomeEmail(42));   // MessageBusInterface
```

- Le type du message est déduit du **type-hint** du premier argument ; sinon `#[AsMessageHandler(handles: Foo::class)]`.
- `#[AsMessageHandler]` s'applique à une **classe** (méthode `__invoke` par défaut, ou `method:`) **ou à une méthode** ; il est répétable. Options réelles (vérifiées dans `vendor/symfony/messenger/Attribute/AsMessageHandler.php`) : `bus`, `fromTransport`, `handles`, `method`, `priority`, `sign`.
- `dispatch()` retourne toujours une **`Envelope`** (message + stamps).
- Un message peut avoir **plusieurs handlers** (événement) ou **un seul** (commande/requête). Les handlers sont trouvés par `HandlersLocator` sur la classe du message, ses parents, ses interfaces (et jokers) : un handler typé sur une interface reçoit tous les messages qui l'implémentent.

### Récupérer un résultat : `HandleTrait` (bus de type query)

```php
class QueryBus
{
    use HandleTrait;
    public function __construct(MessageBusInterface $messageBus) { $this->messageBus = $messageBus; }
    public function ask(object $query): mixed { return $this->handle($query); }   // lit le HandledStamp
}
```
`handle()` lève `LogicException` si **zéro** ou **plusieurs** handlers ont répondu. Le résultat vient du `HandledStamp`. Un message routé vers un transport async n'est pas traité au dispatch : pas de résultat.

## 2. Enveloppe et stamps

`Envelope` = enveloppe **immuable** contenant un message + des stamps ; le message lui-même n’est pas rendu immuable (`with()` retourne une nouvelle enveloppe ; `last(Foo::class)`, `all()`, `withoutAll()`).

Stamps à connaître (dossier `Stamp/`) :

| Stamp | Rôle |
|-------|------|
| `BusNameStamp` | nom du bus d'origine (ajouté par `add_bus_name_stamp_middleware`) |
| `DelayStamp` | retarder la livraison (`DelayStamp::delayFor(new \DateInterval('PT10M'))`, `delayUntil()`) |
| `TransportNamesStamp` | forcer le(s) transport(s) pour ce dispatch |
| `HandledStamp` | résultat + nom du handler (ajouté après traitement) |
| `SentStamp` | message envoyé sur un transport |
| `ReceivedStamp` | message reçu depuis un transport (par un worker) |
| `RedeliveryStamp` | compteur de retries |
| `ErrorDetailsStamp` | exception ayant causé l'échec |
| `SentToFailureTransportStamp` | message envoyé au failure transport |
| `DispatchAfterCurrentBusStamp` | traiter après la fin du message courant |
| `ValidationStamp` | groupes de validation (middleware `validation`) |
| `SerializerStamp` | contexte du serializer de transport |
| `DeduplicateStamp` | (7.3+, nécessite `symfony/lock`) éviter les doublons |

```php
$bus->dispatch(new Report($id), [new DelayStamp(5000), new TransportNamesStamp('async')]);
// ou directement : $bus->dispatch(Envelope::wrap($msg)->with(new DelayStamp(5000)));
```

Les stamps implémentant `NonSendableStampInterface` (ex. `HandledStamp`, `ReceivedStamp`) ne sont **pas sérialisés** vers le transport.

## 3. Bus multiples et middleware

```yaml
framework:
    messenger:
        default_bus: command.bus          # OBLIGATOIRE dès qu'on définit plus d'un bus
        buses:
            command.bus:
                middleware: [validation]                # doctrine_transaction requiert une intégration Doctrine absente ici   # ajoutés AVANT les middleware par défaut "send/handle"
            query.bus: ~
            event.bus:
                default_middleware:
                    allow_no_handlers: true                      # un événement sans handler n'est pas une erreur
```

- Un bus = un service portant son nom configuré (`command.bus`, ou `messenger.bus.default` par défaut) ; autowiring par nom d'argument (`MessageBusInterface $commandBus`) ou `#[Target('command.bus')]`.
- Ordre des **middleware** (vérifié dans `FrameworkExtension`, section messenger) : défauts « before » (`add_default_stamps` → `add_bus_name_stamp` → `reject_redelivered_message` → `dispatch_after_current_bus` → `decode_failed_message` (8.1) → `failed_message_processing` → *deduplicate si symfony/lock*) → **tes middleware** → `send_message` → `handle_message`.
- `send_message` : s'il y a au moins un sender pour le message (et que ce n'est pas un message *reçu* d'un worker), il envoie et **ne passe pas** au middleware suivant (donc pas de `handle_message`). Sans sender, il appelle le suivant. `handle_message` appelle les handlers. Par défaut `allow_no_senders: true`, `allow_no_handlers: false` (exception `NoHandlerForMessageException`).
- Middleware custom :

```php
final class AuditMiddleware implements MiddlewareInterface
{
    public function handle(Envelope $envelope, StackInterface $stack): Envelope
    {
        // avant
        $envelope = $stack->next()->handle($envelope, $stack);   // oublier cet appel = court-circuiter la chaîne
        // après
        return $envelope;
    }
}
```
Déclaration : `middleware: [App\Messenger\AuditMiddleware]` (service autowiré).
- Le middleware tourne **aussi côté worker** (le message reçu est re-dispatché sur le bus, détecté via `ReceivedStamp`).
- `dispatch_after_current_bus` : un message dispatché *depuis un handler* avec `new DispatchAfterCurrentBusStamp()` n'est traité qu'après la fin du handler courant (utile pour les événements post-commit).

## 4. Transports et routage

```yaml
framework:
    messenger:
        transports:
            async:
                dsn: '%env(MESSENGER_TRANSPORT_DSN)%'     # doctrine://default, redis://..., amqp://...
                retry_strategy: { max_retries: 3, delay: 1000, multiplier: 2, max_delay: 0, jitter: 0.1 }
            failed: 'doctrine://default?queue_name=failed'
            sync: 'sync://'                                # handler exécuté dans le dispatch, sans worker
        failure_transport: failed
        routing:
            App\Message\SendWelcomeEmail: async
            App\Message\ImportantEvent: [async, audit]     # plusieurs transports
            'App\Message\*': async                         # jokers
```

Alternative au YAML : l'attribut **`#[AsMessage('async')]`** sur la classe du message (7.2+) — `transport` accepte string|array, plus `serializedTypeName`.

- **Sans routage** : le message est traité **immédiatement** (synchrone) par `handle_message`.
- DSN : `sync://`, `in-memory://` (natifs), `doctrine://` / `redis://` / `amqp://` / `sqs://` (packages séparés).
- Sérialisation : par défaut `messenger.transport.native_php_serializer` (config `framework.messenger.serializer.default_serializer`) ; le service `messenger.transport.symfony_serializer` (JSON via le composant Serializer) est à activer par transport (`serializer:`).
- Un message est routé via le **`SendersLocator`** (classe, parents, interfaces).

## 5. Worker, retry, failure transport

```bash
php bin/console messenger:consume async -vv            # worker
php bin/console messenger:consume async --limit=10 --time-limit=3600 --memory-limit=128M
php bin/console messenger:stop-workers                 # arrêt gracieux (signal via cache) -> redéploiement
```
Options réelles de `messenger:consume` : `--limit`, `--failure-limit`, `--memory-limit`, `--time-limit`, `--sleep`, `--bus`, `--queues`, `--no-reset`, `--all`, `--exclude-receivers`, `--keepalive`, `--fetch-size` (8.1). Le worker écoute SIGTERM/SIGINT/SIGQUIT/SIGALRM (extension `pcntl`).

**Retry** (`MultiplierRetryStrategy`, valeurs par défaut vérifiées) : `max_retries: 3`, `delay: 1000` ms, `multiplier: 2`, `max_delay: 0` (illimité), `jitter: 0.1`.
Délai = `delay * multiplier ^ nbRetries`, ± jitter. Avec les défauts : ~1 s, ~2 s, ~4 s. `max_retries: 0` désactive les retries. Stratégie perso : `retry_strategy: { service: App\...\MyStrategy }` (implémente `RetryStrategyInterface`, incompatible avec les autres clés).

Contrôler le retry depuis un handler :
- `UnrecoverableMessageHandlingException` : **jamais** de retry → direct au failure transport.
- `RecoverableMessageHandlingException` : force les retries au-delà de `max_retries` en 8.0 déjà ; `getRetryDelay()` permet de préciser le délai. **8.1 uniquement** : le paramètre `forceRetry: false` et la méthode `forceRetry()` permettent de respecter à nouveau la limite configurée.
- Ces exceptions peuvent être **encapsulées** dans une `HandlerFailedException` : Messenger inspecte les exceptions imbriquées (`SendFailedMessageForRetryListener`).

**Failure transport** : après épuisement des retries (ou exception irrécupérable), le message part avec `SentToFailureTransportStamp` / `ErrorDetailsStamp` :
```bash
php bin/console messenger:failed:show            # liste
php bin/console messenger:failed:show 12 -vv     # détail
php bin/console messenger:failed:retry           # rejoue (interactif)
php bin/console messenger:failed:remove 12
```
`failure_transport` global, ou **par transport** (`transports.async.failure_transport`). Sans failure transport, le message échoué est **supprimé** après les retries.

**Événements du worker** : `WorkerMessageReceivedEvent`, `WorkerMessageHandledEvent`, `WorkerMessageFailedEvent` (`willRetry()`), `WorkerMessageRetriedEvent`, `WorkerRunningEvent`, `WorkerStartedEvent`, `WorkerStoppedEvent`. Côté bus : `SendMessageToTransportsEvent`, `MessageSentToTransportsEvent` (7.4).

## 6. Tester

- Transport `in-memory://` dans `when@test` : `async: 'in-memory://'`. Récupérer `static::getContainer()->get('messenger.transport.async')` (`InMemoryTransport`) puis `->getSent()`, `->getAcknowledged()`, `->getRejected()`, `->reset()`.
- Pour tester un handler : l'instancier directement (c'est une classe PHP ordinaire).
- Pour tester un middleware : `MiddlewareTestCase` (`Symfony\Component\Messenger\Test\Middleware`) fournit `getStackMock()`.

## Ce qui change vs Symfony 6/7
- `MessageHandlerInterface` / `MessageSubscriberInterface` : **supprimées** (7.0). Seul `#[AsMessageHandler]` existe.
- `#[AsMessage]` (7.2) : routage depuis le message lui-même.
- 7.3 : `DeduplicateMiddleware`/`DeduplicateStamp`, `SentForRetryStamp`. 7.4 : `AddDefaultStampsMiddleware`, `MessageSentToTransportsEvent`, signature des messages par handler.
- 8.0 : `RecoverableExceptionInterface::getRetryDelay()`. 8.1 : `forceRetry()`, `DecodeFailedMessageMiddleware`, `--fetch-size`, regex dans les noms de transports de `messenger:consume`, `StopWorkerOnTimeLimitListener` déprécié au profit de l'option `time_limit`.
- 8.1 : un échec de **décodage** ne supprime plus le message : il suit le chemin retry/failure normal.

## Piège certification
- Sans sender sélectionné (ni `routing`, ni `#[AsMessage]`, ni `TransportNamesStamp`), le traitement est **synchrone** : pas de worker nécessaire, rien dans une file.
- Plus d'un bus ⇒ `default_bus` obligatoire (sinon `InvalidConfigurationException`).
- Un handler qui lève une exception est enveloppé dans `HandlerFailedException` (via `getWrappedExceptions()`, `getPrevious()` = première erreur).
- `Envelope::with()` retourne une **nouvelle** enveloppe ; cela ne clone ni le message ni les objets stamps. Utilise des DTO readonly pour éviter leur mutation.
- `UnrecoverableMessageHandlingException` ≠ retry ; `max_retries: 3` = 1 tentative initiale + 3 retries.
- Un worker est un process **long** : il garde le code en mémoire → `messenger:stop-workers` (ou redémarrage) après déploiement ; les services « resettables » sont reset entre messages (sauf `--no-reset`).
- Le message doit être **sérialisable** : pas de ressource, de closure, ni d'entité Doctrine entière (envoyer l'id).

## Observer dans l'app
- `bin/console debug:messenger` : handlers par message, par bus (`debug:messenger --bus=command.bus`).
- `bin/console debug:config framework messenger` et `debug:container --parameter=messenger.bus.default.middleware`.
- `bin/console debug:container messenger.bus.default` ; `debug:autowiring MessageBus`.
- `bin/console messenger:consume async -vv` et `messenger:stats`, `messenger:setup-transports`.
- Après installation facultative du profiler : profiler : onglet **Messenger** (messages dispatchés, bus, handlers, stamps) — alimenté par le middleware `traceable` en debug.
- Tests : `php bin/phpunit tests/Messenger`.

## Doc officielle
- https://symfony.com/doc/current/messenger.html
- https://symfony.com/doc/current/messenger/handler_results.html
- https://symfony.com/doc/current/messenger/multiple_buses.html
- https://symfony.com/doc/current/messenger/custom-transport.html
- https://symfony.com/doc/current/messenger/dispatch_after_current_bus.html

## 7. Ce qu'un test en mémoire prouve réellement

Avec `in-memory://`, le bus envoie une enveloppe et le handler ne tourne pas immédiatement. Le transport conserve les messages dans l'objet PHP du processus courant. Lancer ensuite `messenger:consume async` dans **un autre terminal** crée un autre processus et une autre mémoire : il ne verra pas les messages du test. Un redémarrage perd également cette file.

Trois niveaux d'observation sont donc distincts :

| Niveau | Ce qu'il prouve | Ce qu'il ne prouve pas |
|---|---|---|
| Appel direct du handler | règle métier et effets sur ses dépendances | routage, ack, retries |
| Dispatch avec `in-memory://` | routage, enveloppe envoyée, absence de traitement immédiat | exécution par un processus séparé |
| `Worker` dans le même processus, listeners branchés | consommation, événements, ack/reject, retry/failure configurés | durabilité ou comportement d'un vrai broker |

Redis ou AMQP sont des prolongements facultatifs : package bridge (`composer require symfony/redis-messenger` ou `symfony/amqp-messenger`), extension PHP correspondante et serveur disponible. Une commande CLI réussie n'est pas une preuve de consommation : suivre un identifiant de message du dispatch à l'effet final. Les commandes `messenger:stats` dépendent aussi des capacités du transport.

### Pourquoi un message peut être traité deux fois

Le worker applique l'effet métier, puis acquitte le message. S'il s'arrête entre les deux, le broker peut relivrer le message. Un retry après une réponse réseau perdue peut produire le même problème. « Un handler et un message » ne veut donc pas dire « un effet exactement une fois ».

```php
final readonly class ReserveSeat
{
    public function __construct(
        public string $operationId,
        public string $seat,
    ) {}
}
// Deux livraisons peuvent porter le même operationId.
```

Définir l'idempotence sur **l'effet observable** : recevoir deux fois `operationId = "booking-42"` doit laisser une seule réservation. Un tableau de clés déjà vues est une simulation pédagogique dans un processus ; il ne protège ni contre un redémarrage ni contre deux workers concurrents. En production, l'enregistrement de l'opération et l'effet doivent être atomiques dans le stockage concerné, ou la dépendance appelée doit proposer sa propre clé d'idempotence. Aucun stockage supplémentaire n'est imposé dans ce module. `DeduplicateStamp` concerne la déduplication via Lock, composant absent et hors du socle de certification retenu ; ce n'est pas une preuve générale d'idempotence métier.

### Un worker ne reconstruit pas tous les services après chaque message

Un service partagé conserve ses propriétés entre invocations. Une propriété `currentTenant` ou un tableau de fichiers temporaires peut donc contaminer le traitement suivant. Éviter l'état mutable lorsque possible ; sinon `Symfony\Contracts\Service\ResetInterface::reset()` permet à l'intégration FrameworkBundle d'enregistrer un nettoyage. Le reset n'est pas une transaction : il ne compense pas un email déjà envoyé. Il ne concerne que les services enregistrés pour cette opération, et `--no-reset` la désactive.

Observer le même objet handler deux fois fait apparaître l'état retenu. Pour prouver le reset de l'intégration, utiliser le `Worker` et son `ResetServicesListener` (ou un worker CLI avec transport partagé) ; deux dispatchs synchrones ne déclenchent pas ce cycle. Les événements utiles : `WorkerStartedEvent`, `WorkerMessageReceivedEvent`, puis `WorkerMessageHandledEvent` ou `WorkerMessageFailedEvent`, enfin `WorkerRunningEvent` et `WorkerStoppedEvent`. Sur l'échec, `willRetry()` dépend des listeners déjà exécutés : journaliser à une priorité après le listener de retry.

### Reprendre après un échec

Un message au failure transport n'est pas réparé par son simple déplacement. Lire sa cause, corriger la dépendance ou le contrat qui échoue, puis rejouer **une fois** en observant l'effet et la clé d'opération. `messenger:failed:remove` abandonne une tentative ; cela ne réalise pas la tâche. Avec `max_retries: 2`, une exception ordinaire produit trois tentatives au total. Une exception récupérable forcée échappe à cette limite ; une irrécupérable ne la consomme pas.

Pour une observation reproductible sans attente, définir `delay: 0`, `jitter: 0` dans le scénario de test. Ne pas prendre ce réglage comme une recommandation de production. Lire `Worker.php`, `SendFailedMessageForRetryListener.php`, `SendFailedMessageToFailureTransportListener.php` et `ResetServicesListener.php` ; [Messenger 8.0](https://symfony.com/doc/8.0/messenger.html).
