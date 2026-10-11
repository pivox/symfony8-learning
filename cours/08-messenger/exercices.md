# Exercices — Module 08

[Parcours](../README.md) · [Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

Code dans `src/Messenger/` (namespace `App\Messenger`), tests dans `tests/Messenger/` (namespace `App\Tests\Messenger`) : `php bin/phpunit tests/Messenger`. Écris le test d'abord.

## 1. Message + handler + dispatch synchrone
Crée un message `RegisterUser` (readonly, avec un email) et son handler annoté `#[AsMessageHandler]`. Le handler stocke l'email dans un petit service `UserRegistry` (tableau en mémoire).
- Test (KernelTestCase) : récupérer `MessageBusInterface`, dispatcher, puis vérifier le contenu de `UserRegistry`.
- Critère : `bin/console debug:messenger` affiche ton handler.
- Question : pourquoi le handler s'exécute-t-il sans worker alors qu'aucun `routing` n'est défini ?

## 2. Lire les stamps de l'enveloppe
Dispatche un message et inspecte l'`Envelope` retournée.
- Test : présence d'un `HandledStamp` (et ce que contient `getResult()`), d'un `BusNameStamp` avec le nom attendu ; absence de `SentStamp`.
- Réflexion : quel middleware ajoute chacun de ces stamps ?

## 3. Bus de requêtes avec `HandleTrait`
Crée un `QueryBus` (classe à toi) basé sur `HandleTrait` et un message `GetUserCount` dont le handler renvoie un entier.
- Test : le résultat est renvoyé ; un message **sans handler** ⇒ exception ; deux handlers pour le même message ⇒ `LogicException`.
- Question : pourquoi cette approche est-elle incompatible avec un transport asynchrone ?

## 4. Trois bus (command / query / event)
Configure `command.bus`, `query.bus`, `event.bus` dans `config/packages/messenger.yaml` (ou un fichier dédié). L'`event.bus` doit tolérer l'absence de handler.
- Test : dispatcher un événement sans handler sur `event.bus` ne lève rien ; le même sur `command.bus` lève `NoHandlerForMessageException`.
- Critère : `debug:messenger --bus=...` et `debug:autowiring MessageBus` montrent les trois bus. Retire `default_bus` et lis l'erreur.
- Question : comment un handler est-il restreint à un seul bus ?

## 5. Middleware personnalisé
Écris `TimingMiddleware` qui ajoute un stamp personnalisé (`TimingStamp` implémentant `StampInterface`, contenant la durée) après l'exécution du reste de la chaîne. Branche-le sur `command.bus`.
- Test unitaire avec `MiddlewareTestCase` (`getStackMock()`), puis un test d'intégration.
- Réflexion : que se passe-t-il si tu n'appelles pas `$stack->next()` ? Où se place ton middleware par rapport à `send_message` / `handle_message` (vérifie via `debug:container --parameter=...middleware`) ?

## 6. Routage asynchrone testable
Ajoute un transport `async` en `in-memory://` pour l'environnement `test`, et route un message `SendNewsletter` vers lui (par `routing` **puis** par `#[AsMessage]`).
- Test : après `dispatch`, le handler n'a **pas** tourné ; `InMemoryTransport::getSent()` contient 1 enveloppe avec un `SentStamp` ; simuler le worker (récupérer l'enveloppe avec `get()` et la redispatcher avec un `ReceivedStamp`) ⇒ handler exécuté.
- Réflexion : quels stamps sont sérialisés vers le transport, lesquels ne le sont pas ?

## 7. Retry et exceptions contrôlées
Écris un handler `FlakyHandler` qui échoue N fois. Configure un transport avec `retry_strategy` (max_retries, delay, multiplier) et un failure transport `failed` (in-memory en test).
- Tests : (a) une `UnrecoverableMessageHandlingException` ⇒ aucun retry, message dans `failed` ; (b) une exception ordinaire ⇒ `max_retries` retries puis `failed` ; (c) test unitaire de `MultiplierRetryStrategy::getWaitingTime()` avec `jitter: 0` pour valider 1000/2000/4000 ms.
- Question : qu'ajoute `forceRetry()` (8.1) à `RecoverableMessageHandlingException` ?

## 8. Observation d'un cycle complet (pas de code, ou script)
Avec un Worker et un transport en mémoire dans le même processus, ou en lisant le code de `SendFailedMessageForRetryListener`, trace le parcours d'un message qui échoue : quels stamps et événements (`WorkerMessageFailedEvent`, `WorkerMessageRetriedEvent`…) interviennent ? Écris-le avec tes mots.

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.

## Critères complémentaires pour les exercices 1–8

Appliquer la [méthode commune](../README.md). Le transport en mémoire et un appel direct au bus n'exécutent pas à eux seuls les listeners de retry d'un worker.

| Exercice | Acceptation supplémentaire |
|---|---|
| 1 | Après un dispatch synchrone : un seul email enregistré ; exception du handler observable par l'appelant. |
| 2 | Utiliser un message sans sender pour attendre l'absence de `SentStamp` ; `sync://` explicitement sélectionné peut en ajouter un. |
| 3 | Zéro handler : avec le bus standard, `NoHandlerForMessageException` peut arriver avant la vérification de `HandleTrait` ; avec un bus tolérant zéro handler, le trait lève `LogicException`. |
| 4 | Handler réservé au query bus absent du traitement du command bus ; vérifier le résultat et pas seulement les tags. |
| 5 | Le middleware appelle le suivant exactement une fois et préserve le résultat ; mesurer une durée positive ou nulle sans seuil instable. |
| 6 | Une enveloppe envoyée, aucun effet immédiat ; après réception simulée, un effet. Le redispatch avec `ReceivedStamp` ne simule ni ack ni retry. |
| 7 | Utiliser `Worker` et listeners de retry/failure dans le même processus ou un vrai transport partagé ; un simple dispatch ne suffit pas. Délais nuls pour le cycle, stratégie seule avec jitter nul pour les calculs. |
| 8 | Le parcours de base reste sans Doctrine : lire les listeners ou utiliser un Worker en mémoire. Tout essai CLI séparé demande un transport partagé réellement disponible. |

## 9. Parcours guidé — observer un worker

Router `GenerateReceipt` vers un transport de test en mémoire. Instancier un `Worker` sur **cette même instance** et arrêter après un message traité (listener d'arrêt approprié).

- Acceptation : avant consommation, `getSent()` compte 1 et aucun reçu produit ; après consommation, un reçu, une enveloppe acquittée et zéro rejet.
- Trace attendue : received avant handled ; démarrage et arrêt présents. Les événements running peuvent se répéter : ne pas imposer un nombre global fragile.
- Indice : `Worker` accepte bus, receivers et event dispatcher ; les attributs ne branchent pas automatiquement des listeners sur un dispatcher construit à la main.

## 10. Parcours intermédiaire — doublons et effet idempotent

Ajouter un `operationId` à `GenerateReceipt`, un registre de test en mémoire et un faux émetteur de reçu qui compte ses appels. Livrer deux enveloppes indépendantes de même identifiant.

- Acceptation : deux livraisons traitées, un seul reçu émis ; deux identifiants différents → deux reçus ; un échec avant émission ne doit pas marquer définitivement l'opération comme réussie.
- Produire une note de trois phrases expliquant pourquoi ce test ne prouve pas la sécurité entre processus ni la résistance au crash après émission.
- Indice : ce que tu comptes est l'effet métier, pas le nombre d'appels au handler.

## 11. Parcours avancé — état retenu et reprise

(a) Faire traiter successivement deux clients par un service partagé conservant un état temporaire ; reproduire puis empêcher la contamination, avec `ResetInterface` si l'état ne peut pas être supprimé.

(b) Faire échouer un message ordinaire avec `max_retries: 2`, délais nuls et failure transport en mémoire. Brancher réellement les listeners nécessaires au Worker.

- Acceptation (a) : le second traitement ne contient aucune donnée du premier ; prouver que le même processus a traité les deux messages et que le mécanisme de reset est exercé.
- Acceptation (b) : trois tentatives, une enveloppe en échec final ; une exception irrécupérable ne provoque qu'une tentative. Après correction du faux service, rejouer l'enveloppe échouée via le chemin approprié produit exactement un effet.
- Indice : observer `willRetry()` après le listener de retry ; lire `failed_message_processing` pour le rejeu, sans copier arbitrairement tous les stamps.

## Défi final — génération de rapports résistante aux relivraisons

Assembler dispatch, réception, génération idempotente, diagnostic d'échec et reprise. Exécuter le scénario de base avec un Worker dans le même processus et les transports en mémoire ; tout essai CLI séparé reste un prolongement avec bridge/broker documentés.

Livrables : trace message/stamps/événements, tests des effets et une procédure de reprise. Acceptation : aucun effet avant réception, doublon sans duplication de rapport, échec final conservé après le nombre prévu de tentatives, traitement suivant sans état résiduel, rejeu corrigé réussi. Expliquer ce qui serait perdu après redémarrage de cette simulation et ce qu'un stockage durable devrait garantir. Indice final : valider le cycle sans erreur avant d'injecter un doublon puis une panne.

## 12. Faire survivre un message au transport — intermédiaire

Durée indicative : 40–55 min. Prérequis : enveloppes, Serializer et [contrat de transport](approfondissement.md).
Scénario : `ExportRequested` contient `reference = 'export-17'` et `format = 'csv'` ; de vieux messages ne possèdent pas encore `format`.
Construire un serializer de transport JSON avec le Serializer de l'application, puis exercer `encode()` et `decode()` dans un `KernelTestCase`.
Ajouter à l'enveloppe un `DelayStamp(250)` et un `ReceivedStamp('training')` ; aucun worker ni broker nécessaire.

- Réussite : le corps encodé contient les deux valeurs ; le retour reconstruit la classe et les valeurs, avec une instance distincte.
- Réussite : délai de 250 conservé et `ReceivedStamp` absent après décodage ; une ancienne fixture de corps `{"reference":"export-17"}` reconstruit `format = 'csv'` grâce au contrat choisi.
- Cas négatif : corps `{"reference":` avec header `type` correct → échec observable, aucun appel au handler métier.
- Version : en 8.1, vérifier le message d'erreur contenu dans l'enveloppe retournée ; expliquer l'exception directe de `decode()` en 8.0 dans la note de test.
- Preuve : distinguer test du contrat de transport et simple `dispatch()` sur un transport mémoire sans sérialisation.
- Indice 1 : les deux `SerializerInterface` n'ont pas le même namespace ni le même objet d'entrée.
- Indice 2 : examiner `NonSendableStampInterface`, le header `type` et les paramètres de constructeur du DTO ; ne pas coder un parseur JSON de transport maison.

## 13. Reporter une notification jusqu'au succès courant — avancé

Durée indicative : 45–60 min. Prérequis : middleware, exceptions de handlers et [dispatch imbriqué](approfondissement.md).
Scénario : `PrepareExport('export-21')` écrit `prepare:start`, dispatche `NotifyExport('export-21')`, puis écrit `prepare:end` dans une trace de test.
Le handler de notification écrit `notify`. Utiliser un seul bus synchrone avec le middleware adéquat et aucune route vers un transport.
Exécuter chaque variante sur une trace remise à zéro ; injecter des pannes déterministes via des dépendances de test.

- Réussite : sans stamp différé → trace exacte `prepare:start, notify, prepare:end` ; avec stamp → `prepare:start, prepare:end, notify`.
- Cas négatif A : le handler principal lève après le dispatch différé → `notify` absent et erreur du handler observable par l'appelant.
- Cas négatif B : le principal réussit mais la notification lève avant d'écrire `notify` → trace `prepare:start, prepare:end` conservée et `DelayedMessageHandlingException` à l'appel racine.
- Preuve : assertions sur un vrai appel à `MessageBusInterface::dispatch()` ; expliquer pourquoi le second échec ne défait pas les effets du premier handler.
- Indice 1 : la décision concerne la fin du dispatch courant, pas un délai en millisecondes.
- Indice 2 : comparer la branche d'échec racine et la boucle de messages différés de `DispatchAfterCurrentBusMiddleware`.
