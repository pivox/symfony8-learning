# Quiz — Module 19 : Mailer

[Cours](README.md) · [Exercices](exercices.md)

Questions d'entraînement originales, socle Symfony 8.0.

**Q1 (une réponse).** Mime et Mailer servent respectivement à :
A. Authentifier et autoriser B. Créer des routes et du HTML C. Représenter et acheminer des messages D. Créer deux bases

**Q2 (une réponse).** Quel type injecter pour demander un envoi ?
A. `MailerInterface` B. `Email` C. `Response` D. `RequestStack`

**Q3 (une réponse).** Le transport `null://null` :
A. Vérifie une boîte réelle B. Échoue toujours C. Envoie via Gmail D. Ne livre pas à un destinataire externe

**Q4 (une réponse).** Un DSN de mailer décrit :
A. Le sujet B. Le transport C. La langue D. Le prénom du destinataire

**Q5 (une réponse).** Un `TemplatedEmail` nouvellement configuré possède-t-il forcément son HTML final ?
A. Oui B. Oui si Twig est installé C. Non D. Oui s'il a un sujet

**Q6 (une réponse).** Dans un contexte destiné à la file, mieux vaut transmettre :
A. Des valeurs ou DTO sérialisables B. Une ressource ouverte C. Une closure D. Le Kernel entier

**Q7 (une réponse).** Installer Messenger rend-il tout email asynchrone ?
A. Oui B. Oui uniquement en prod C. Oui avec Twig D. Non, le bus et son routage comptent

**Q8 (une réponse).** Une mise en file réussie prouve :
A. La lecture par le destinataire B. L'acceptation dans le chemin de file testé C. L'absence future d'erreur SMTP D. L'absence de spam

**Q9 (une réponse).** Une file `in-memory://` du test est accessible par un autre processus worker ?
A. Oui B. Oui avec le même nom C. Non D. Oui après un `sleep`

**Q10 (une réponse).** Une erreur technique d'acheminement implémente :
A. `TransportExceptionInterface` B. `TranslatorInterface` C. `UserInterface` D. `FormTypeInterface`

**Q11 (une réponse).** `MailerInterface::send()` retourne :
A. Le nombre de lectures B. Une garantie de livraison C. Un booléen D. `void`

**Q12 (une réponse).** Quel contrôle vise explicitement la mise en file ?
A. `assertResponseIsSuccessful()` B. `assertQueuedEmailCount()` C. `assertCount()` sur les routes D. `lint:twig`

**Q13 (une réponse).** Un handler rejoué peut-il envoyer deux fois ?
A. Non B. Seulement sans Twig C. Oui, sans stratégie adaptée D. Non avec une adresse valide

**Q14 (une réponse).** Pour comparer enveloppe et en-têtes, il faut retenir :
A. Acheminement et représentation visible ont des rôles distincts B. Ils sont toujours identiques C. Bcc doit être visible de tous D. L'enveloppe contient le HTML

**Q15 (plusieurs réponses).** Quelles assertions renforcent un test d'envoi ?
A. Une attente de cinq secondes B. Une connexion externe obligatoire C. Le nombre de messages attendu D. Destinataire, sujet et corps exacts

## Réponses

1. **C** — Les autres propositions n'ont pas de lien avec la représentation MIME et l'acheminement du mail.
2. **A** — Le message est une donnée ; requête et pile de requêtes ne sont pas des services d'envoi.
3. **D** — Le transport nul permet un chemin local sans livraison réelle ; ce n'est ni une panne simulée ni un fournisseur.
4. **B** — Destinataires, sujet et langue appartiennent au message ou au rendu, pas à l'adresse du transport.
5. **C** — Le rendu peut survenir plus tard ; installation de Twig et présence du sujet ne forcent pas cette étape.
6. **A** — Les autres objets portent des états ou ressources inadaptés à une sérialisation fiable.
7. **D** — La présence du composant n'impose pas le routage asynchrone ; environnement et template ne suffisent pas non plus.
8. **B** — Les étapes suivantes peuvent échouer. Ni lecture ni délivrabilité ne sont garanties par cette preuve locale.
9. **C** — Deux processus ne partagent pas cette mémoire ; nom de transport et attente ne créent pas de persistance.
10. **A** — Les autres interfaces concernent traduction, sécurité et formulaires.
11. **D** — Le contrat ne renvoie pas de booléen, de métrique de lecture ou de preuve de livraison.
12. **B** — Une réponse HTTP réussie et un template valide ne démontrent pas qu'un email a été mis en file.
13. **C** — Une reprise peut répéter un effet ; Twig et la validité d'une adresse ne dédupliquent pas les envois.
14. **A** — Les destinataires de l'enveloppe et les en-têtes visibles répondent à des objectifs différents, notamment pour Bcc.
15. **C, D** — Les comptes vérifient la quantité, le contenu vérifie le contrat. Attente réelle et réseau ne rendent pas ces assertions meilleures.
