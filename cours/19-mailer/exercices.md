# Exercices — Module 19

[Cours](README.md) · [Quiz](quiz.md) · [Méthode](../README.md)

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
