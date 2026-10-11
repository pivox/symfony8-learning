# Exercices — Module 19

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md) · [Méthode](../README.md)

Installer Mailer selon le cours. Code dans `src/Mailer/`, tests dans `tests/Mailer/`, templates dans `templates/emails/`. Transport nul, adresses sous `example.test`, aucun envoi externe. Tests de services : `KernelTestCase` ; HTTP : `WebTestCase`.

## 1. Guidé — message exact

Crée une factory pour une confirmation d'inscription à un atelier. Entrées : adresse, nom et titre de l'atelier ; aucune persistance. Expéditeur `formation@example.test`, sujet `Inscription confirmée`, corps texte mentionnant le nom et le titre.

- **Critères :** destinataire et expéditeur exacts, un seul destinataire, sujet exact, texte attendu ; une adresse mal formée est refusée. Aucun appel au mailer dans la factory.
- **Indice :** teste les propriétés du message avant de réfléchir au transport.

## 2. Guidé — deux templates

Ajoute un corps HTML et un template texte explicite. Le HTML affiche le titre de l'atelier dans un titre, puis salue la personne.

- **Critères :** les deux corps sont rendus ; le nom `<b>Sam</b>` apparaît comme donnée échappée dans le HTML, pas comme une balise ajoutée par l'utilisateur ; le texte reste lisible.
- **Indice :** la simple construction de `TemplatedEmail` ne prouve pas que le rendu a eu lieu.

## 3. Autonome — pièce jointe

Joins une fiche texte `atelier.txt` contenant le titre de l'atelier, générée pour le test dans un répertoire temporaire.

- **Critères :** une pièce jointe, nom et contenu attendus, type `text/plain` ; nettoyage après le test. Un chemin absent provoque une erreur explicite au moment où son contenu est requis, jamais un envoi silencieux sans fiche.
- **Indice :** vérifie le contenu effectivement lu ; certaines lectures sont différées.

## 4. Autonome — envoi synchrone observable

Crée `RegistrationNotifier` injectant le mailer et la factory. Dans le contexte de test synchrone, utilise le transport nul et désactive le bus du mailer.

- **Critères :** un appel au service produit un email observé ; zéro email mis en file ; sujet, destinataire et corps vérifiés. Aucun accès SMTP.
- **Indice :** associe compte d'emails et assertions de contenu ; un compteur seul ne prouve pas quel message a été envoyé.

## 5. Diagnostic — transport en échec

Substitue un transport qui lève une exception implémentant `TransportExceptionInterface`. Le service la traduit en `NotificationUnavailable`, classe métier à créer.

- **Critères :** exception métier attendue, cause conservée et absence de succès annoncé. Le chemin reste synchrone pour que l'échec soit observable dans l'appel testé.
- **Indice :** un transport nul réussit ; il ne suffit pas à représenter ce scénario.

## 6. Autonome — mise en file

Dans un contexte de test distinct, active le bus du mailer et route `SendEmailMessage` vers `mail_async`, transport `in-memory://`.

- **Critères :** un email mis en file, une enveloppe contenant le bon message et aucun passage au transport d'acheminement avant traitement. Inspecte le destinataire et le contexte sérialisable.
- **Indice :** ne lance pas un worker séparé pour lire une file qui n'existe que dans la mémoire du test.

## 7. Diagnostic — traiter l'enveloppe

Récupère le message de l'exercice 6 et fais-le traiter par le handler d'envoi, avec un transport de test. Ajoute une variante où ce transport échoue.

- **Critères :** un acheminement simulé après traitement ; échec levé par le handler dans la variante. La mise en file initiale demeure un événement distinct. Aucun nouveau cycle de mise en file involontaire.
- **Indice :** le handler Mailer s'appuie sur `TransportInterface` ; rappelle-toi le rôle de `ReceivedStamp` si tu choisis de passer par le bus.

## 8. Défi final — confirmation depuis HTTP

Crée une action publique de laboratoire `POST /training/registrations`, avec DTO validé via `MapRequestPayload` : nom, email et titre d'atelier obligatoires. Pas de compte utilisateur ni de stockage. Utilise le contexte asynchrone de l'exercice 6.

- **Critères :** JSON valide → 202 et un email mis en file ; email invalide ou nom vide → 422 et aucune mise en file ; JSON malformé → 400 ; puis test séparé du traitement et du contenu final. Les files sont réinitialisées entre cas.
- **Livrable :** tests et explication de ce que signifie 202, sans prétendre confirmer une livraison réelle.
- **Indice :** les assertions sur la requête initiale et sur le traitement différé répondent à deux questions différentes.

## 9. Autonome — distinguer destinataire visible et destinataire technique

**Niveau :** intermédiaire · **Durée indicative :** 50 minutes.
**Prérequis :** Mailer installé via Composer/Flex, exercices 1 et 4 ; [enveloppe et copie cachée](approfondissement.md).

Scénario : une confirmation affiche un destinataire et deux copies, mais un laboratoire doit pouvoir l'acheminer vers un collecteur unique.
Construis un email texte avec `From=formation@example.test`, `To=ada@example.test`, `Cc=equipe@example.test`, `Bcc=archive@example.test` et sujet `Atelier R-42`.
Inspecte d'abord l'enveloppe dérivée ; envoie ensuite via le mailer synchrone nul avec une enveloppe explicite d'expéditeur `rebonds@example.test` et de seul destinataire `collecteur@example.test`.
Le test reste local, avec un dispatcher permettant d'observer le `MessageEvent` réel.

- **Critères :** l'enveloppe dérivée contient les trois destinataires attendus ; l'enveloppe observée lors de l'envoi explicite contient uniquement le collecteur et l'expéditeur technique choisi.
- **Critères :** les en-têtes visibles gardent `To=ada@example.test`, `Cc=equipe@example.test` et `From=formation@example.test` ; le contenu n'est pas réécrit par le changement d'enveloppe.
- **Cas négatif :** les en-têtes préparés par `getPreparedHeaders()` ne contiennent aucun champ `Bcc`, alors que l'objet initial garde bien `archive@example.test` dans `getBcc()`.
- **Livrable :** assertions séparées sur l'objet de composition, l'enveloppe et les en-têtes MIME ; un seul appel synchrone au mailer, aucune mise en file.
- **Indice 1 :** une adresse utilisée pour acheminer le message n'a pas nécessairement le même rôle qu'une adresse affichée dans ses en-têtes.
- **Indice 2 :** compare `Envelope::create($email)`, le second argument de `send()` et `Message::getPreparedHeaders()` ; ne cherche pas une preuve de copie cachée dans le seul compteur d'emails.

## 10. Défi — refuser un envoi avant le transport

**Niveau :** avancé · **Durée indicative :** 65 minutes.
**Prérequis :** Mailer installé, événements avec `AsEventListener`, exercice 4 et [cycle de refus](approfondissement.md).

Scénario : un listener de laboratoire bloque les emails dont l'en-tête `X-Training-State` vaut `blocked`, et laisse passer ceux marqués `allowed`.
Les messages ont tous un expéditeur `formation@example.test`, un destinataire `ada@example.test`, un sujet `Contrôle R-42` et un corps texte complet.
Garde le bus du mailer désactivé ; utilise un transport espion dérivé d'`AbstractTransport` dont `doSend()` incrémente un compteur, sans réseau.
Branche le dispatcher réel au transport et observe séparément `MessageEvent`, `SentMessageEvent` et `FailedMessageEvent`.

- **Critères :** un appel réel au mailer avec `allowed` produit un appel à `doSend()`, un événement sent et aucun événement failed.
- **Critères :** un appel indépendant avec `blocked` déclenche le listener de refus, marque son événement rejeté, produit zéro `doSend()`, zéro sent et zéro failed, sans exception de transport.
- **Cas négatif :** retirer ou mal enregistrer le listener fait échouer le test `blocked` ; un transport espion qui court-circuite tout le dispatcher ne satisfait pas l'exercice.
- **Livrable :** compteurs réinitialisés entre cas et explication du résultat de `send(): void` après refus ; ne présente pas ce retour comme un accusé de livraison.
- **Indice 1 :** un refus de politique et une tentative de transport échouée occupent deux branches différentes du cycle.
- **Indice 2 :** `reject()` arrête aussi la propagation ; capture l'état depuis le listener de refus ou un observateur exécuté avant lui, et vérifie les priorités avec `debug:event-dispatcher`.
