# Atelier 04 — De l’acceptation HTTP à l’effet d’un message

[Ateliers](README.md) · [Messenger](../08-messenger/README.md) · [Serializer](../07-serializer/README.md) · [Mailer](../19-mailer/README.md)

**Niveau :** avancé. **Durée :** 4–5 h. **Prérequis :** modules 03, 04, 07, 08, 12 et 14. Code dans `src/Ateliers/Async/`, tests dans `tests/Ateliers/Async/`.

## Mission et périmètre

Crée une API publique de laboratoire qui accepte une demande de rapport, la place dans une file puis fait traiter le message par un handler. L’effet est une notification enregistrée par une dépendance simulée en mémoire. Aucun email réel, broker ou stockage métier n’est nécessaire.

La route `POST /training/report-jobs` reçoit :

```json
{"requestId":"report-001","email":"sam@example.test","topic":"http"}
```

`requestId` suit `report-` puis trois chiffres ; `email` est obligatoire et valide ; `topic` vaut `http` ou `symfony`. Le message est un DTO `readonly` contenant uniquement ces trois chaînes. Le DTO d’entrée et le message peuvent être distincts pour garder le transport indépendant du contrat HTTP.

Le succès HTTP retourne **202** avec `requestId` et `status: "accepted"`. Il signifie uniquement que le dispatch a réussi dans le processus. Avec une file en mémoire, cela ne garantit ni durabilité après arrêt, ni traitement, ni livraison.

## Jalon 1 — Rejeter avant de dispatcher

Lie les données via `#[MapRequestPayload]`, contraintes Validator et route au format JSON. Accepte uniquement JSON. Le contrôleur délègue la construction du message et le dispatch à un service.

| Entrée | Statut | Effet |
|---|---|---|
| Exemple valide | 202 | Une enveloppe envoyée |
| `email` invalide, `requestId` invalide ou `topic` inconnu | 422 | Aucune enveloppe |
| Corps `{"requestId":` | 400 | Aucune enveloppe |
| Corps texte avec `Content-Type: text/plain` | 415 | Aucune enveloppe |
| Dispatch qui lève une exception d’indisponibilité simulée | 503 selon le contrat de l’atelier | Pas de faux 202 |

La conversion de l’indisponibilité en 503 est un comportement à implémenter, pas un résultat par défaut universel de Messenger. Garde la cause dans l’exception métier. Vérifie les effets sur un transport réinitialisé à chaque scénario.

## Jalon 2 — Vérifier le passage par la sérialisation

Route le message vers un transport dédié `training_reports` en `in-memory://?serialize=true`, avec `messenger.transport.symfony_serializer`. La sérialisation réelle du transport doit être activée pour ne pas tester uniquement le passage d’une référence PHP entre deux objets.

Après dispatch, récupère l’enveloppe et vérifie type du message, égalité des trois valeurs et absence d’effet métier immédiat. Le message décodé doit être une autre instance avec les mêmes données. Ne mets ni service, ni ressource de fichier, ni objet HTTP dans le message.

**Validation :** un envoi, aucun appel à la dépendance de notification, valeurs exactes après aller-retour. Lancer un worker dans un autre terminal ne pourrait pas voir cette file : elle appartient au processus du test.

Observe dans `InMemoryTransportFactory` le traitement de `serialize`, puis dans `InMemoryTransport` l’endroit où encode/decode est appelé. Une telle simulation ne valide pas les limites d’un broker réel.

## Jalon 3 — Traiter et rendre l’effet observable

Le handler enregistré avec `#[AsMessageHandler]` prépare un résultat `requestId`, `recipient`, `subject`. Pour `topic=http`, le sujet vaut `Rapport HTTP` ; pour `symfony`, `Rapport Symfony`. Il appelle une interface de notification remplacée par un faux qui mémorise les appels dans le test.

Traite l’enveloppe dans le même processus. Si tu redispatches sur le bus, indique qu’elle provient du transport afin de ne pas la renvoyer indéfiniment dans la file. Une invocation directe du handler convient pour tester la règle métier, mais ne prouve pas l’acquittement d’un worker.

**Validation :** après traitement, un seul appel avec sujet et destinataire exacts ; la file n’a pas reçu une nouvelle copie par erreur. Dans un autre scénario, la dépendance lève une exception et le handler ne prétend pas avoir réussi.

Réalise ensuite une expérience de double livraison du même message. Avec le handler de base, deux livraisons produisent deux appels. Écris cette observation avant de proposer une extension d’idempotence : un `requestId` seul ne déduplique rien.

## Jalon 4 — Retry du worker, dans un test distinct

Prépare un `Worker` dans le même processus avec les listeners de retry/failure et un transport d’échec dédiés. Fixe `max_retries: 2`, `delay: 0`, `jitter: 0` pour ce scénario uniquement. Borne le worker par un mécanisme d’arrêt explicite pour ne pas bloquer les tests.

Dans un premier test, la dépendance échoue deux fois puis réussit. Dans un second, elle échoue toujours. Utilise une exception ordinaire, pas une exception récupérable qui force des retries au-delà de la limite.

**Validation :** trois tentatives dans chaque cas ; un effet final dans le premier, zéro dans le second ; une enveloppe dans le failure transport seulement dans le second. Observe le compteur du faux et le message d’échec. Ne remplace pas le worker par trois appels manuels au handler en affirmant avoir testé Messenger.

Si ce jalon est trop avancé, rends les trois premiers et indique explicitement que retry/failure restent non validés. Le module 08 décrit les listeners et les événements utiles.

## Incident à diagnostiquer

L’API retourne 202, le test trouve un message envoyé, mais aucun effet n’apparaît. Écris une chronologie « HTTP → dispatch → sérialisation → consommation → effet ». Place tes assertions sur cette chronologie et explique pourquoi un test du premier segment ne prouve pas les suivants.

Deuxième défaut : redispatcher sans le stamp de réception remet le message dans la file. Reproduis une seule itération, constate l’envoi supplémentaire puis corrige. Évite une boucle sans limite dans le test de diagnostic.

## Indices progressifs

<details>
<summary>Indice 1 — Lire l’enveloppe</summary>

Le bus choisit un sender avant le handler. Un message reçu d’un transport ne suit pas le même chemin qu’un nouveau dispatch. Regarde les stamps et les traces de dépendances avant de changer le routage.

</details>

<details>
<summary>Indice 2 — Source locale et listeners</summary>

Lis `SendMessageMiddleware`, `ReceivedStamp`, `Worker`, `SendFailedMessageForRetryListener` et `SendFailedMessageToFailureTransportListener`. Pour le test du worker, tous les listeners nécessaires doivent être branchés ; instancier `Worker` seul ne recopie pas toute la configuration FrameworkBundle.

</details>

## Extensions facultatives

**Idempotence :** fais mémoriser les IDs déjà traités dans un faux de notification et vérifie un effet pour deux appels. Présente-le comme une simulation limitée à ce processus. Explique séparément la panne « effet appliqué puis arrêt avant mémorisation » ; une vraie garantie exige un mécanisme atomique adapté au stockage ou une clé d’idempotence acceptée par le fournisseur.

**Mailer :** après `composer require symfony/mailer` via Flex, remplace la notification simulée par une implémentation qui prépare un email vers `example.test`, avec transport nul et mailer synchrone dans ce handler. Ce choix évite d’ajouter involontairement une deuxième file. Vérifie contenu et destinataire selon le module 19 ; le transport nul ne prouve pas une livraison dans une boîte.

## Livraison personnelle

- [ ] Matrice HTTP et absence d’envoi sur entrée invalide.
- [ ] Aller-retour sérialisé et absence d’effet avant traitement.
- [ ] Effet du handler et double livraison observés.
- [ ] Retry/failure testés avec un worker, ou jalon explicitement non réalisé.
- [ ] Limites de durabilité et d’idempotence expliquées.

Références : [Messenger et tests](https://symfony.com/doc/8.0/messenger.html#in-memory-transport), [Serializer](https://symfony.com/doc/8.0/serializer.html), [Mailer](https://symfony.com/doc/8.0/mailer.html). Les handlers, stamps et factories installés permettent de vérifier le comportement précis de la version du projet.
