# Approfondissement — contrat de transport et dispatch imbriqué

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Le cours distingue déjà envoi, consommation et retry.
Cette fiche examine ce qui traverse effectivement un transport, puis l'ordre des messages dispatchés depuis un handler.
Socle **8.0**, vérifié avec **8.1.8** ; le changement de décodage propre à 8.1 est signalé séparément.
Prérequis : Envelope, stamps, middleware et test d'un bus dans le même processus.
Aucun broker, Doctrine ou composant Lock n'est requis.

## 1. Une enveloppe PHP n'est pas encore un contrat de transport

Un transport doit pouvoir conserver ou transmettre une représentation du message.
Le serializer de transport Messenger transforme une `Envelope` en tableau contenant `body` et `headers`.
Ce rôle diffère de `Symfony\Component\Serializer\SerializerInterface`, qui sérialise des données applicatives.
Le serializer JSON de Messenger s'appuie justement sur ce composant pour produire le corps et les stamps transmissibles.
Le header `type` permet au consommateur de retrouver la classe attendue.
Renommer une classe encore référencée par des messages en attente peut donc casser leur lecture.

```php
use Symfony\Component\Messenger\Envelope;
use Symfony\Component\Messenger\Transport\Serialization\Serializer;

$transportSerializer = new Serializer($applicationSerializer);
$wire = $transportSerializer->encode(new Envelope($message));
$restored = $transportSerializer->decode($wire);
```

Ici `$applicationSerializer` est le service Serializer de l'application ; `$message` est un DTO de l'atelier.
Comparer la classe et les valeurs après retour, sans attendre l'identité d'instance.
Les stamps `NonSendableStampInterface` sont retirés pendant l'encodage.
Par exemple, `ReceivedStamp` exprime une réception dans le processus courant et ne doit pas voyager comme une nouvelle instruction.
Un `DelayStamp` est au contraire transmissible ; comparer son délai après décodage.
Un résultat `HandledStamp` ne constitue pas une réponse distante attachée automatiquement au message transmis.

### Tester réellement la frontière

Un `InMemoryTransport` sans serializer conserve les enveloppes PHP : il peut masquer un contrat impossible à encoder.
Son constructeur accepte un serializer de transport pour exercer la conversion sans broker.
Avec FrameworkBundle, le DSN `in-memory://?serialize=true` permet aussi de l'activer.
Un test `encode()` puis `decode()` est encore plus ciblé pour constater le contenu envoyé.
Un succès du handler appelé directement ne prouve donc pas la compatibilité d'un message avec un transport.

| Vérification | Donnée déterministe | Résultat à observer |
|---|---|---|
| corps du message | référence `report-17`, format `csv` | valeurs et types conservés |
| stamp transmissible | délai 250 ms | `DelayStamp` retrouvé |
| stamp local | `ReceivedStamp('training')` | absent après le retour |
| ancien corps | champ optionnel absent | défaut documenté du nouveau DTO |
| corps illisible | JSON tronqué | échec de décodage, pas exécution métier |

Pour faire évoluer le contrat, conserver une fixture représentant un ancien corps réellement accepté.
Un nouveau paramètre obligatoire sans défaut peut rendre cette fixture impossible à reconstruire.
Un défaut explicite peut assurer une compatibilité d'entrée, mais il doit être correct métier.
Un champ `schemaVersion` peut guider une adaptation ; sa seule présence ne transforme aucune donnée.
Éviter de transporter un service, une ressource ou un graphe vivant pour demander un traitement futur.

**Différence de version :** en 8.0, le serializer JSON de transport peut lever `MessageDecodingFailedException` directement.
En **8.1 uniquement**, `Messenger\Transport\Serialization\Serializer`, utilisé ici avec JSON, peut retourner une enveloppe portant cette exception comme message.
`DecodeFailedMessageMiddleware` rétablit ensuite le chemin d'échec dans le traitement du bus.
Un test direct de `decode()` en 8.1 doit donc examiner le message retourné, pas seulement attendre une exception.
Les tests des exercices doivent indiquer explicitement quelle version ils ciblent.

## 2. Reporter un message après le dispatch courant

Un handler A qui dispatche B sans routage asynchrone provoque normalement l'exécution imbriquée de B.
Si A note `début`, dispatche B, puis note `fin`, la trace peut être `début, B, fin`.
`DispatchAfterCurrentBusStamp` demande au middleware de mettre B en attente jusqu'au succès du dispatch courant.
La trace devient alors `début, fin, B`, si aucun autre middleware ne change ce scénario.
Il s'agit d'un ordre d'exécution dans le processus ; cela ne crée ni file durable ni tâche indépendante.

```php
use Symfony\Component\Messenger\Stamp\DispatchAfterCurrentBusStamp;

// Fragment appelé depuis un handler ; le middleware doit être présent sur le bus.
$bus->dispatch($notification, [new DispatchAfterCurrentBusStamp()]);
```

Si A échoue après avoir mis B en attente, le middleware abandonne ce message différé.
Si A réussit puis B échoue, A a déjà terminé et ses effets ne sont pas annulés.
L'appel racine peut alors lever `DelayedMessageHandlingException`.
Il faut distinguer l'exception visible par l'appelant de l'état métier laissé par A.
Sans stockage transactionnel, une trace ou un faux émetteur ne dispose d'aucun rollback magique.

| Scénario | Ordre ou effet à vérifier | Ce qu'il ne prouve pas |
|---|---|---|
| B sans stamp | B s'exécute pendant A | aucun découplage dans le temps |
| B avec stamp, A réussit | B après la fin de A | aucune durabilité |
| B avec stamp, A échoue | B n'est pas traité | les effets précédents de A peuvent subsister |
| A réussit, B échoue | échec différé observable | aucune compensation automatique de A |

Pour plusieurs bus, le middleware doit être disponible selon le montage documenté ; commencer avec un seul bus.
Un `MessageBus` assemblé à la main n'hérite pas des middleware configurés par FrameworkBundle.
La liste exacte des middleware constitue donc une partie du scénario testé.

## 3. Observer dans l'application

```bash
symfony console debug:config framework messenger
symfony console debug:container messenger.transport.symfony_serializer
symfony console debug:container --parameter=messenger.bus.default.middleware
symfony console debug:messenger
php bin/phpunit tests/Messenger
```

Conserver la trace dans un service de test dédié et vérifier l'ordre complet, sans temporisation réelle.
Pour le transport, afficher dans le diagnostic les noms des headers et les valeurs fictives, pas de vrais secrets.
Les fixtures de contrat passent par les services réellement sélectionnés : le serializer natif PHP n'est pas le serializer JSON.
Lire les exceptions imbriquées lorsque le bus encapsule une erreur de handler.

## Sources officielles et code installé

- [Messenger 8.0 : sérialisation et messages imbriqués](https://symfony.com/doc/8.0/messenger.html).
- [Messenger 8.1 : évolution des erreurs de décodage](https://symfony.com/doc/8.1/messenger.html).
- `vendor/symfony/messenger/Transport/Serialization/Serializer.php` : corps, headers, filtrage des stamps et erreurs.
- `vendor/symfony/messenger/Transport/InMemory/InMemoryTransport.php` : présence ou absence du serializer.
- `vendor/symfony/messenger/Middleware/DispatchAfterCurrentBusMiddleware.php` : file locale et abandon sur échec.

Relier chaque assertion à sa frontière : valeur encodée, ordre de dispatch ou effet métier.
