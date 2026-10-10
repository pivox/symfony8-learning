# Module 19 — Mailer

[Parcours](../README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Construire un email vérifiable, séparer rendu et transport, comprendre l'envoi via Messenger et tester les échecs. Prérequis : [Mime](../10-composants/README.md), [Twig](../09-twig-forms/README.md), [Messenger](../08-messenger/README.md), [tests](../14-tests-diagnostic/README.md).

**Versions :** application Symfony 8.1.8 ; API de base du module disponible en Symfony 8.0. Aucun fournisseur tiers, compte SMTP réel ni destinataire réel n'est nécessaire.

## Préparation au moment de pratiquer

Mime et Twig sont présents, Mailer est absent dans le dépôt initial. Installer la capacité avec Flex :

```bash
composer require symfony/mailer
```

Laisser appliquer la recette, puis choisir un transport `null://null` pour le laboratoire et les tests. Il accepte les messages sans livraison externe. Les tests peuvent observer les événements de Mailer ; ce transport ne simule pas les réponses d'un serveur SMTP.

Le DSN décrit un transport, pas le destinataire. Tout futur identifiant réel doit rester dans `.env.local` ou le coffre ; la configuration versionnée garde uniquement des valeurs d'exemple.

## 1. Trois responsabilités

| Responsabilité | Outil | Question à tester |
|---|---|---|
| Construction | `Email`, `Address`, `DataPart` | Qui reçoit quoi, avec quels fichiers ? |
| Rendu | `TemplatedEmail`, Twig | Texte exact et échappement des paramètres |
| Acheminement | `MailerInterface`, transport | Envoi demandé, différé ou en échec ? |

Mime sait représenter un email ; il ne suffit pas à le livrer. Une factory peut construire le message, tandis qu'un service injectant `Symfony\Component\Mailer\MailerInterface` déclenche son envoi. Le contrôleur délègue et traduit le résultat pour l'utilisateur.

## 2. Structure d'un message

Un message peut avoir corps texte et HTML, destinataires `To`, `Cc`, `Bcc`, réponse `Reply-To` et pièces jointes. Le sujet et les destinataires font partie du contrat, autant que le corps.

L'enveloppe d'acheminement et les en-têtes visibles ne sont pas la même représentation. Ne déduis pas que l'adresse d'un destinataire caché doit apparaître dans les en-têtes du message reçu par les autres.

Les assertions peuvent vérifier les adresses, le sujet, les corps et les pièces jointes avant tout accès réseau. Une fixture de pièce jointe est copiée dans un dossier temporaire si le test doit la modifier.

## 3. Rendu Twig

`Symfony\Bridge\Twig\Mime\TemplatedEmail` associe un template et un contexte. Un exemple minimal de paramétrage, sans service d'envoi :

```php
use Symfony\Bridge\Twig\Mime\TemplatedEmail;

$email = (new TemplatedEmail())
    ->htmlTemplate('emails/example.html.twig')
    ->textTemplate('emails/example.txt.twig')
    ->context(['name' => 'Sam']);
```

Ce fragment ne forme pas un message envoyable à lui seul : expéditeur, destinataire et sujet restent à définir. Le HTML rendu n'est pas nécessairement présent immédiatement après construction ; teste le rendu via le renderer ou le chemin d'envoi configuré.

Garde dans le contexte des valeurs simples ou des DTO adaptés. Une ressource ouverte ou une closure n'est pas un bon contenu pour un message destiné à être sérialisé. Utilise un template texte explicite pour maîtriser sa formulation.

## 4. Envoi synchrone et différé

`MailerInterface::send()` retourne `void` : ce n'est pas une preuve de lecture dans une boîte mail. Sans bus, Mailer délègue au transport. Avec le bus configuré, il distribue un `SendEmailMessage` ; le routage Messenger détermine si ce message sera traité immédiatement ou envoyé à un transport asynchrone.

**Installer Messenger ne suffit pas à rendre les emails asynchrones.** Observe le routage et le bus. En différé, un succès au moment de la requête prouve au mieux la mise en file ; le worker peut rencontrer une erreur plus tard. [Envoi et configuration Mailer](https://symfony.com/doc/current/mailer.html).

Pour le laboratoire de file, `in-memory://` permet d'inspecter l'enveloppe dans le processus de test. Il ne transmet pas cette mémoire à un worker CLI lancé dans un autre processus. Le test de traitement doit appeler le handler ou redistribuer l'enveloppe avec le contexte de réception approprié ; voir le module Messenger.

## 5. Événements et échecs

Une erreur d'acheminement expose `TransportExceptionInterface`. L'application peut la traduire en erreur métier et conserver sa cause. Pour un email différé, cette décision intervient au traitement, pas forcément dans le contrôleur initial.

`MessageEvent`, `SentMessageEvent` et `FailedMessageEvent` permettent d'observer le cycle. Un `MessageEvent` peut être associé à la mise en file : examine `isQueued()` avant d'interpréter l'événement comme une livraison au transport.

Le worker pouvant rejouer un traitement, ne promets pas une livraison exactement une fois. Le défi reste une preuve de mise en file et de traitement local ; une déduplication durable demanderait un autre périmètre.

## 6. Tests observables

Les assertions Mailer du FrameworkBundle distinguent les événements d'envoi et ceux marqués pour le passage par le bus. Selon le scénario, utilise `assertEmailCount()`, `assertQueuedEmailCount()` puis `getMailerMessage()` et les assertions sur le contenu. Le compteur « queued » repose sur `MessageEvent::isQueued()`, émis avant le dispatch : seul, il ne prouve pas qu'un transport asynchrone a stocké l'enveloppe. Vérifie aussi celle-ci dans le transport de test et l'absence d'acheminement avant traitement.

En test synchrone, désactive explicitement le bus du mailer (`framework.mailer.message_bus: false`) si tu veux tester uniquement le chemin direct. En test de mise en file, configure un bus et un routage vers le transport de test. Chaque configuration doit annoncer ce qu'elle prouve.

Un transport nul ne prouve ni livraison, ni passage d'un filtre antispam, ni affichage dans tous les logiciels de messagerie. Les tests vérifient le contrat de l'application, pas un réseau absent.

## Observer et sources

Après installation :

```bash
php bin/console debug:config framework mailer
php bin/console debug:autowiring MailerInterface
php bin/console debug:messenger
php bin/console debug:event-dispatcher 'Symfony\Component\Mailer\Event\MessageEvent'
php bin/phpunit tests/Mailer
```

- [Mailer 8.0 : chemin d'envoi](https://github.com/symfony/mailer/blob/v8.0.0/Mailer.php).
- [Handler d'envoi 8.0](https://github.com/symfony/mailer/blob/v8.0.0/Messenger/MessageHandler.php).
- Déjà consultable : `vendor/symfony/framework-bundle/Test/MailerAssertionsTrait.php`.
