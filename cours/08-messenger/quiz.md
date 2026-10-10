# Quiz — Module 08 : Messenger

Réponds sans regarder la doc. Une ou plusieurs bonnes réponses : c'est indiqué. Réponses en bas.

**Q1.** (1 réponse) Sans routing, attribut `AsMessage` ni `TransportNamesStamp` sélectionnant un transport, où est traité un message dispatché ?
- A. Il est ignoré
- B. Dans la file par défaut `default`
- C. Immédiatement, de façon synchrone, par `HandleMessageMiddleware`
- D. Il lève `NoSenderForMessageException`

**Q2.** (1 réponse) Que retourne `MessageBusInterface::dispatch()` ?
- A. Le résultat du handler
- B. `void`
- C. Une `Envelope`
- D. Le message

**Q3.** (2 réponses) Parmi ces éléments, lesquels ont été supprimés de Messenger (7.0) ?
- A. `MessageHandlerInterface`
- B. `#[AsMessageHandler]`
- C. `MessageSubscriberInterface`
- D. `MessageBusInterface`

**Q4.** (1 réponse) Tu définis deux bus. Que faut-il obligatoirement ajouter ?
- A. `default_transport`
- B. `default_bus`
- C. `allow_no_handlers: true`
- D. Rien, le premier est pris par défaut

**Q5.** (1 réponse) Quel stamp permet de retarder la livraison d'un message ?
- A. `DelayStamp`
- B. `TransportNamesStamp`
- C. `RedeliveryStamp`
- D. `SleepStamp`

**Q6.** (2 réponses) Valeurs par défaut de la `retry_strategy` d'un transport :
- A. `max_retries: 3`
- B. `delay: 1000`
- C. `multiplier: 1`
- D. `jitter: 0`

**Q7.** (1 réponse) Quel est l'effet de `UnrecoverableMessageHandlingException` ?
- A. Le message est retenté indéfiniment
- B. Le message n'est pas retenté ; il part au failure transport s'il y en a un
- C. Le worker s'arrête
- D. Le message est rejoué en synchrone

**Q8.** (1 réponse) Un middleware n'appelle pas `$stack->next()->handle(...)`. Conséquence ?
- A. Rien de spécial, Messenger appelle la suite
- B. La chaîne est interrompue : le message n'atteint pas les middleware suivants (ni les handlers)
- C. Une exception est levée automatiquement
- D. Le message est envoyé au failure transport

**Q9.** (1 réponse) Un message est routé vers `async`. Que fait `SendMessageMiddleware` ?
- A. Envoie sur le transport puis laisse `HandleMessageMiddleware` traiter aussi
- B. Envoie sur le transport et n'appelle pas la suite de la chaîne
- C. Ignore le routage si le message est valide
- D. Traite le message puis l'envoie

**Q10.** (2 réponses) Options valides de `#[AsMessageHandler]` :
- A. `bus`
- B. `fromTransport`
- C. `queue`
- D. `async`

**Q11.** (1 réponse) Quel est le rôle de `HandleTrait::handle()` ?
- A. Dispatche et retourne le résultat du handler ; lève `LogicException` si 0 ou plusieurs handlers
- B. Dispatche en asynchrone
- C. Ajoute un `DelayStamp`
- D. Consomme un transport

**Q12.** (1 réponse) Sans `failure_transport` configuré, un message qui épuise ses retries :
- A. Reste dans la file
- B. Est supprimé
- C. Est rejoué indéfiniment
- D. Est envoyé à `sync`

**Q13.** (1 réponse) Quelle commande rejoue les messages en échec ?
- A. `messenger:consume --retry`
- B. `messenger:failed:retry`
- C. `messenger:replay`
- D. `messenger:setup-transports`

**Q14.** (2 réponses) Comment tester sans broker externe ?
- A. DSN `in-memory://` + `InMemoryTransport::getSent()`
- B. DSN `sync://` pour exécuter les handlers dans le dispatch
- C. `fake://`
- D. Un broker Redis réel est obligatoire

**Q15.** (1 réponse) À quoi sert `DispatchAfterCurrentBusStamp` ?
- A. Dispatcher à la fin du processus PHP
- B. Retarder un message d'une minute
- C. Traiter le message dispatché depuis un handler seulement après la fin du handler courant
- D. Envoyer le message sur un autre bus

---

## Questions de mise en situation — 16 à 20

**Q16.** (1 réponse) Un test envoie vers `in-memory://`, puis une autre commande PHP démarre un worker. Pourquoi celui-ci ne voit-il pas ce message ?

- A. Le message n’a pas de HandledStamp
- B. Les objets transport n’ont pas une mémoire partagée entre processus
- C. Un worker ne peut traiter que Doctrine
- D. Il manque toujours un DelayStamp

**Q17.** (1 réponse) Un worker applique un effet puis meurt avant l’ack. Quelle propriété faut-il prévoir ?

- A. Aucune nouvelle livraison n’est possible
- B. Le handler est forcément exécuté exactement une fois
- C. Une relivraison possible et un effet métier idempotent
- D. La disparition automatique de tous les effets déjà produits

**Q18.** (1 réponse) Un service partagé garde `currentCustomer` dans une propriété. Que garantit le reset des services du worker ?

- A. Tous les objets sont reconstruits
- B. Les effets métier sont annulés
- C. Les messages en échec sont supprimés
- D. Les services enregistrés pour reset peuvent nettoyer leur état entre traitements

**Q19.** (1 réponse) Avec une exception ordinaire et `max_retries: 2`, combien de tentatives précèdent l’échec final ?

- A. 1
- B. 2
- C. 3
- D. Une infinité

**Q20.** (1 réponse) Quelle action répare réellement une tâche au failure transport ?

- A. La supprimer
- B. Rejouer sans diagnostic autant de fois que possible
- C. Ajouter seulement un HandledStamp
- D. Corriger la cause puis rejouer et vérifier l’effet attendu

## Réponses

1. **C.** Sans sender, `SendMessageMiddleware` appelle le suivant et `HandleMessageMiddleware` traite. `allow_no_senders` vaut `true` par défaut.
2. **C.** Toujours une `Envelope` ; le résultat se lit dans le `HandledStamp` (c'est ce que fait `HandleTrait`).
3. **A et C.** Supprimées en 7.0 (`CHANGELOG`) ; seul `#[AsMessageHandler]` reste.
4. **B.** « You must specify the "default_bus" if you define more than one bus. » (`Configuration.php`).
5. **A.** `DelayStamp` (`delayFor()`, `delayUntil()`). `TransportNamesStamp` choisit le transport, `RedeliveryStamp` compte les retries.
6. **A et B.** `max_retries: 3`, `delay: 1000`, `multiplier: 2` (pas 1), `max_delay: 0`, `jitter: 0.1` (pas 0).
7. **B.** Pas de retry ; failure transport si configuré, sinon le message est abandonné.
8. **B.** Le middleware est responsable de poursuivre la chaîne.
9. **B.** Un message envoyé (avec sender) n'est pas traité localement : « message should only be sent and not be handled ».
10. **A et B.** Options : `bus`, `fromTransport`, `handles`, `method`, `priority`, `sign`.
11. **A.** Voir `HandleTrait` : exceptions si zéro ou plusieurs `HandledStamp`.
12. **B.** Sans failure transport, il est supprimé ; d'où l'intérêt de `failure_transport`.
13. **B.** `messenger:failed:retry` (avec `:show` et `:remove`).
14. **A et B.** `in-memory://` expose `getSent()`, `getAcknowledged()`, `getRejected()` ; `sync://` traite immédiatement. `fake://` n'existe pas.
15. **C.** Utile pour dispatcher des événements « post-traitement » depuis un handler (middleware `dispatch_after_current_bus`).


## Réponses raisonnées — 16 à 20

16. **B.** Le message reste dans le processus du test. A est normal avant traitement ; C exclut à tort les autres transports ; D n’a aucun rôle de partage mémoire.
17. **C.** La séparation effet/ack permet un doublon. A et B supposent une garantie absente ; D supposerait une transaction couvrant une dépendance externe.
18. **D.** Le reset appelle les services concernés ; ce n’est ni une reconstruction générale (A), ni un rollback (B), ni une politique de file (C).
19. **C.** Une tentative initiale plus deux retries. A ignore la stratégie, B oublie la tentative initiale ; D pourrait concerner une exception forçant les retries mais pas ce cas.
20. **D.** Reprise signifie réussite observée. A abandonne, B répète le défaut et peut multiplier les effets ; C simule une information interne sans réaliser le traitement.
