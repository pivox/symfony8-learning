# Approfondissement — enveloppe, copie cachée et refus d'envoi

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Un email possède un contenu visible et une instruction d'acheminement distincte.
Son passage dans un listener n'est pas non plus une preuve que le transport a été appelé.
Ces deux distinctions rendent les tests d'envoi plus précis sans nécessiter de serveur SMTP.
Les API étudiées appartiennent au socle Symfony 8.0 ; le dépôt est en 8.1.8.
Au moment de pratiquer seulement, installer Mailer avec `composer require symfony/mailer` et laisser Flex appliquer sa recette.
Mime et EventDispatcher sont déjà présents ; utiliser des adresses `example.test` et un acheminement local simulé.

## 1. Les en-têtes ne sont pas l'enveloppe

`Email::to()`, `cc()` et `bcc()` décrivent les destinataires dans l'objet de composition.
`Envelope` représente l'expéditeur technique et les destinataires utilisés pour l'acheminement.
Sans enveloppe explicite, Mailer peut la construire depuis le message avec `Envelope::create()`.
Les destinataires dérivés comprennent les adresses `To`, `Cc` et `Bcc`.
Une enveloppe explicite permet de choisir une destination technique différente des en-têtes visibles.
Les rebonds peuvent ainsi avoir une adresse différente de celle affichée comme expéditeur.
L'enveloppe ne réécrit pas automatiquement le contenu du message pour le rendre cohérent avec un autre scénario.

```php
use Symfony\Component\Mailer\Envelope;
use Symfony\Component\Mime\Address;

$envelope = new Envelope(
    new Address('rebonds@example.test'),
    [new Address('collecteur@example.test')],
);
```

Ce fragment décrit une enveloppe de laboratoire ; il ne déclenche aucun envoi.
La passer comme second argument de `MailerInterface::send()` demande cet acheminement particulier.
Le [code Envelope 8.0.0](https://github.com/symfony/mailer/blob/v8.0.0/Envelope.php) et [DelayedEnvelope 8.0.0](https://github.com/symfony/mailer/blob/v8.0.0/DelayedEnvelope.php) expliquent la dérivation.

L'objet `Email` peut conserver `getBcc()` pour construire l'enveloppe et permettre les assertions.
Lors de la préparation des en-têtes MIME, `Message::getPreparedHeaders()` enlève le champ `Bcc`.
Une assertion sur l'objet avant transport et une assertion sur le MIME final observent donc deux représentations différentes.
Vérifie l'absence du **champ** `Bcc` dans les en-têtes préparés, pas l'absence absolue d'une chaîne arbitraire dans tout le corps.
Un corps peut légitimement contenir la même adresse comme texte sans être un en-tête de copie cachée.
La [documentation Mime 8.0](https://symfony.com/doc/8.0/components/mime.html) situe la construction et le rendu du message.

## 2. Un listener peut refuser l'acheminement

`MessageEvent` est un point d'interception avant l'envoi ou avant le passage au bus.
Sa méthode `reject()` marque l'événement comme rejeté et arrête sa propagation.
Le transport basé sur `AbstractTransport` observe ce rejet avant d'appeler `doSend()`.
Dans ce cas, il n'exécute pas l'acheminement et retourne sans produire `SentMessageEvent`.
Ce refus volontaire n'est pas une panne SMTP et ne devient pas automatiquement une exception métier.
`MailerInterface::send()` retourne toujours `void` ; l'appelant n'obtient pas directement le motif de refus.
Si ton cas d'usage doit présenter ce motif à un utilisateur, définis aussi un contrat de résultat explicite au bon niveau.
Le [code AbstractTransport 8.0.0](https://github.com/symfony/mailer/blob/v8.0.0/Transport/AbstractTransport.php) décrit l'ordre des opérations.

```php
// Dans un listener recevant un MessageEvent :
$queued = $event->isQueued();
$rejected = $event->isRejected();
```

Ces deux états répondent à des questions indépendantes : phase de traitement et décision de refus.
Si le mailer utilise un bus, un événement `isQueued() === true` précède le dispatch.
Le listener peut également refuser à cet endroit, avant qu'un `SendEmailMessage` ne soit distribué.
Lors du traitement, le transport émet un autre `MessageEvent`, cette fois sans le marqueur queued.
Une politique doit donc préciser les phases où elle agit pour éviter des effets répétés involontaires.
Les [sources du Mailer 8.0.0](https://github.com/symfony/mailer/blob/v8.0.0/Mailer.php) montrent ce chemin.

## 3. Compter le bon événement

Les assertions Mailer du FrameworkBundle s'appuient sur des `MessageEvent` enregistrés.
Elles ne constituent pas un accusé de réception SMTP ni un compteur universel de livraisons réussies.
Un listener exécuté avant le logger peut arrêter la propagation et empêcher ce logger de voir le message.
Un observateur exécuté avant le refus peut au contraire avoir vu cet événement.
Le nombre d'événements collectés dépend donc aussi des priorités et du point d'observation.
Pour vérifier un refus, observe `isRejected()`, l'absence de `doSend()` et l'absence de `SentMessageEvent`.
Pour vérifier une panne pendant `doSend()`, observe l'exception et, avec le dispatcher branché, `FailedMessageEvent`.

| Situation | Acheminement exécuté | Observation pertinente |
|---|---|---|
| Enveloppe explicite | Selon chemin d'envoi | Destinataires techniques, en-têtes séparés |
| Copie cachée | Destinataire dans l'enveloppe dérivée | Pas de champ `Bcc` dans les en-têtes préparés |
| Rejet avant transport | Non | État rejeté, zéro `doSend()`, zéro événement sent |
| Transport local réussi | Oui, simulé | Appel transport et `SentMessageEvent` |
| Échec dans le transport | Tentative échouée | Exception et `FailedMessageEvent` |

## Observer après installation

```bash
php bin/console debug:config framework mailer
php bin/console debug:event-dispatcher 'Symfony\Component\Mailer\Event\MessageEvent'
php bin/console debug:event-dispatcher 'Symfony\Component\Mailer\Event\SentMessageEvent'
php bin/phpunit tests/Mailer
```

Le laboratoire de refus reste synchrone : `framework.mailer.message_bus: false` dans son contexte de test.
Utilise un transport espion dérivé d'`AbstractTransport` avec le dispatcher réel pour garder le cycle d'événements.
Remplacer tout le transport par un objet qui ne dispatch rien supprimerait justement le comportement étudié.
Sources déjà locales : `vendor/symfony/mime/Message.php` et `vendor/symfony/framework-bundle/Test/MailerAssertionsTrait.php`.
Après installation : `vendor/symfony/mailer/Event/MessageEvent.php` et `Transport/AbstractTransport.php`.
Le [contrat de MessageEvent 8.0.0](https://github.com/symfony/mailer/blob/v8.0.0/Event/MessageEvent.php) précise l'arrêt de propagation.
Les [exercices 9 et 10](exercices.md) permettent de vérifier chaque représentation et chaque point d'observation.
