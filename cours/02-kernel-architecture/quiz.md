# Quiz — Module 02 : Kernel & architecture

Une seule bonne réponse sauf mention *(plusieurs réponses)*. Réponses en bas.

---

**Q1.** Que retourne `public/index.php` dans une application Symfony 8 avec le composant Runtime ?
- A. Un objet `Response` déjà envoyé
- B. Un callable (closure) qui retourne le Kernel
- C. `void`, l'application est lancée par `$kernel->run()`
- D. Le `Container`

**Q2.** Quel est l'ordre correct des événements pour une requête réussie dont le controller retourne un tableau (non `Response`) et qui a un listener `kernel.view` ?
- A. request, controller, controller_arguments, view, response, finish_request
- B. request, controller_arguments, controller, view, response
- C. request, controller, view, controller_arguments, response, terminate
- D. request, controller, controller_arguments, response, view, finish_request

**Q3.** Quand `kernel.view` est-il dispatché ?
- A. Toujours
- B. Uniquement si le controller ne retourne pas une instance de `Response`
- C. Uniquement pour les requêtes au format HTML
- D. Uniquement en mode debug

**Q4.** Quand `kernel.terminate` est-il dispatché ?
- A. À la fin de `HttpKernel::handle()`
- B. Par `HttpKernel::terminate()`, appelé par le Runner après l'envoi de la réponse
- C. Avant `kernel.response`
- D. Seulement quand une exception survient

**Q5.** Un listener `kernel.request` appelle `$event->setResponse($response)`. Que se passe-t-il ?
- A. Le controller est quand même exécuté
- B. La propagation est stoppée, le controller n'est pas appelé, et la réponse passe par `kernel.response`
- C. Une exception est levée
- D. La réponse est ignorée

**Q6.** Un listener A a la priorité 10 et un listener B la priorité -5 sur le même événement. Lequel s'exécute en premier ?
- A. B
- B. A
- C. L'ordre de déclaration
- D. Ils s'exécutent en parallèle

**Q7.** Quelle est la priorité par défaut d'un listener déclaré avec `#[AsEventListener]` sans `priority` ?
- A. -1
- B. 0
- C. 10
- D. 255

**Q8.** Dans le framework, quel listener est exécuté en premier pour `kernel.request` (priorité la plus haute parmi ces choix) ?
- A. `RouterListener` (32)
- B. `LocaleListener::onKernelRequest` (16)
- C. `ValidateRequestListener` (256)
- D. `FragmentListener` (48)

**Q9.** Pourquoi `LocaleListener::onKernelRequest` (priorité 16) est-il après `RouterListener` (32) ?
- A. Pour pouvoir lire l'attribut `_locale` posé par le routing
- B. Parce que les locales sont chargées par Twig
- C. Pour réinitialiser la session
- D. Aucune raison particulière

**Q10.** Qu'est-ce qu'une sous-requête ? *(plusieurs réponses)*
- A. Un appel à `HttpKernel::handle()` avec `HttpKernelInterface::SUB_REQUEST`
- B. Ce que fait `AbstractController::forward()`
- C. Une requête qui ne déclenche pas d'événements `kernel.*`
- D. Une requête pour laquelle `$event->isMainRequest()` vaut `false`

**Q11.** Que fait un `ValueResolverInterface::resolve()` qui ne peut pas traiter un argument ?
- A. Lève une exception
- B. Retourne `null`
- C. Retourne un itérable vide, pour laisser les autres resolvers essayer
- D. Retourne `false`

**Q12.** Quel resolver a la priorité la plus **basse** (s'exécute en dernier) parmi ceux-ci ?
- A. `RequestValueResolver`
- B. `ServiceValueResolver`
- C. `DefaultValueResolver`
- D. `BackedEnumValueResolver`

**Q13.** Quelle est l'API à utiliser pour accéder à la requête courante dans un service ?
- A. Injecter `Request` directement
- B. Injecter `RequestStack` et appeler `getCurrentRequest()` / `getMainRequest()`
- C. Utiliser `$_SERVER`
- D. Appeler `Kernel::getRequest()`

**Q14.** Que se passe-t-il quand un controller ne retourne rien (`null`) et qu'aucun listener `kernel.view` ne pose de réponse ?
- A. Réponse 204
- B. Réponse 200 vide
- C. `ControllerDoesNotReturnResponseException`
- D. Une `Response` est créée depuis `null` par défaut

**Q15.** Que fait l'attribut `#[Serialize]` (nouveauté 8.1) ?
- A. Sérialise la session
- B. Sérialise la valeur retournée par le controller en réponse, via `kernel.view`
- C. Met en cache la réponse
- D. Valide le payload entrant

**Q16 (bonus).** Quel est le rôle de `HttpKernel::handle(..., catch: false)` ?
- A. Désactive les événements
- B. Ne convertit pas les exceptions en réponse : elles sont relancées (après `kernel.finish_request`)
- C. Désactive le profiler
- D. Active le mode sous-requête

---


## Questions complémentaires — mise en situation

Une seule bonne réponse par question.

**Q17.** À quel moment la requête courante est-elle dépilée dans HttpKernel::handle() ?
- A. Avant kernel.request
- B. Dans le finally, après le traitement
- C. Seulement après un statut 200
- D. Dans tous les listeners finish_request

**Q18.** Pendant une sous-requête, getMainRequest() retourne quoi ?
- A. La sous-requête
- B. Toujours null
- C. La requête principale
- D. La prochaine requête du navigateur

**Q19.** ResponseEvent::setResponse() arrête-t-il automatiquement la propagation ?
- A. Oui, comme RequestEvent
- B. Non : il remplace la réponse
- C. Il lève toujours une exception
- D. Il termine immédiatement le processus

**Q20.** Une exception métier est convertie en 409 par ton listener kernel.exception. Quelle suite normale attend-on ?
- A. Aucun événement supplémentaire
- B. kernel.response puis kernel.finish_request
- C. Retour au contrôleur qui a échoué
- D. kernel.terminate dans handle()

**Q21.** Pourquoi kernel.terminate ne suffit-il pas pour garantir une notification ?
- A. Il crée automatiquement une transaction distribuée
- B. Il réexécute automatiquement après panne
- C. Il reste synchrone dans le processus et ne fournit pas de reprise
- D. Il est réservé aux erreurs 500

## Réponses

**Q1 — B.** Le script retourne un callable qui reçoit `$context` et retourne le Kernel ; `autoload_runtime.php` choisit le Runner.

**Q2 — A.** Séquence de `handleRaw()` : `request` → `controller` → `controller_arguments` → (appel) → `view` → `response` → `finish_request`. `terminate` est hors de `handle()`.

**Q3 — B.** `ViewEvent` n'est dispatché que si le retour n'est pas une `Response`.

**Q4 — B.** Dans `HttpKernelRunner`, après `send()` et `fastcgi_finish_request()` (hors debug), `$kernel->terminate()` dispatche `TerminateEvent`.

**Q5 — B.** `RequestEvent::setResponse()` stoppe la propagation ; le controller est sauté, mais la réponse passe par `filterResponse()` (donc `kernel.response` et `kernel.finish_request`).

**Q6 — B.** Plus la priorité est élevée, plus tôt s'exécute le listener.

**Q7 — B.** `priority: 0` par défaut.

**Q8 — C.** `ValidateRequestListener` à 256 (le `DebugHandlersListener` en dev est à 2048 mais n'est pas proposé). Les autres : Router 32, Locale 16, Fragment 48.

**Q9 — A.** « must be registered after the Router to have access to the _locale » (commentaire du code de `LocaleListener`).

**Q10 — A, B, D.** `forward()` fait un `handle()` en `SUB_REQUEST`. Les événements sont dispatchés aussi pour les sous-requêtes (C faux), d'où `isMainRequest()`.

**Q11 — C.** Un resolver retourne un itérable vide ; le premier qui produit une valeur gagne. Si aucun ne résout, `RuntimeException` (argument non résolu).

**Q12 — C.** `DefaultValueResolver` : -100 (`ServiceValueResolver` : -50 ; `RequestValueResolver` : 120 ; `BackedEnumValueResolver` : 100). Seul `VariadicValueResolver` (-150) est plus bas.

**Q13 — B.** On n'injecte pas `Request` dans un service (non partagé dans le container) ; `RequestStack` donne la requête courante/principale/parente.

**Q14 — C.** Le message suggère d'ajouter un `return`.

**Q15 — B.** `#[Serialize]` (HttpKernel 8.1, `code`, `headers`, `context`) : géré par `SerializeControllerResultAttributeListener` sur l'événement `kernel.view`.

**Q16 — B.** Voir `HttpKernel::handle()` : avec `catch: false`, `finishRequest()` est appelé puis l'exception est relancée.

**Q17 — B.** Le finally assure le dépilement même en échec. A empêcherait l’observation pendant la requête, C laisserait fuir l’état en erreur, D confond un événement de nettoyage et le propriétaire de la pile.

**Q18 — C.** Le sommet courant change, la base reste la principale. A décrit getCurrentRequest(), B ne vaut que sans requête principale empilée, D ne fait pas partie de la pile.

**Q19 — B.** Le remplacement ne stoppe pas automatiquement les listeners suivants. A transpose à tort le comportement de RequestEvent ; C et D ne correspondent pas à cette API.

**Q20 — B.** La réponse suit le filtrage normal. A oublie ce filtrage, C suppose une reprise inexistante au point du throw, D confond handle et terminate.

**Q21 — C.** Le hook ne fournit ni file durable ni retry. A et B inventent des garanties, D confond terminaison et traitement d’erreur ; une livraison fiable demande un mécanisme dédié.

