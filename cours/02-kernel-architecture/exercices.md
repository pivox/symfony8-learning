# Exercices — Module 02

Code dans `src/Architecture/` (namespace `App\Architecture`), tests dans `tests/Architecture/` (namespace `App\Tests\Architecture`) : `php bin/phpunit tests/Architecture`. Écris le test d'abord.

## 1. Observer avant de coder
Sans écrire de code : avec `bin/console debug:event-dispatcher kernel.request`, dresse la liste ordonnée des listeners avec leurs priorités. Fais de même pour `kernel.exception` et `kernel.response`.
- Questions : quel listener résout la route ? Où se place la locale par rapport au routing et pourquoi ? Quel listener construit la réponse d'erreur ?

## 2. Kernel en test, sans HTTP
Dans `tests/Architecture/`, écris un test qui construit un `Request::create('/')` et appelle directement `static::bootKernel()->handle($request)` (ou via `KernelBrowser`).
- Test : la requête sur une route inexistante donne un 404 ; avec `catch: false`, une `NotFoundHttpException` est levée.
- Question : que change `catch: false` pour `kernel.exception` et pour `kernel.finish_request` ?

## 3. Premier listener `kernel.response`
Crée `App\Architecture\EventListener\ResponseHeaderListener` qui ajoute un header `X-App-Version` (valeur injectée via paramètre) sur les **requêtes principales uniquement**.
- Test fonctionnel : le header est présent sur une page normale et sur une page 404.
- Question : que se passe-t-il pour une sous-requête (ex. `forward()`) ? Prouve-le avec un second test qui compte les appels.

## 4. Court-circuiter avec `kernel.request`
Crée un listener `kernel.request` qui renvoie directement une `Response` 503 si un fichier `var/maintenance.flag` existe (chemin injecté).
- Tests : avec flag, 503 et le controller n'est **pas** exécuté ; sans flag, comportement normal.
- Réflexion : quelle priorité choisir pour passer avant le routing ? Avant la session ? Justifie avec la sortie de l'exercice 1.

## 5. `kernel.view` : convertir un retour en `Response`
Crée un controller qui retourne un `array` (ou un petit DTO) et un listener `kernel.view` qui produit une `JsonResponse`.
- Tests : le JSON est correct ; si le controller retourne `null` et que ton listener ignore ce cas, l'exception `ControllerDoesNotReturnResponseException` est levée.
- Réflexion : comparer avec l'attribut `#[Serialize]` (8.1) ; quelle différence de configuration ? Lis `SerializeControllerResultAttributeListener` dans `vendor/`.

## 6. Un ValueResolver personnalisé
Crée `App\Architecture\Resolver\CurrentLocaleInfoResolver` (`ValueResolverInterface`) qui injecte dans un controller un petit value object `ClientInfo` (IP, langue préférée via `Request::getPreferredLanguage()`).
- Tests : le controller reçoit un `ClientInfo` correct ; un paramètre d'un autre type n'est pas impacté (le resolver rend la main).
- Questions : où voir le resolver et sa priorité (`debug:container --tag=controller.argument_value_resolver`) ? Que se passe-t-il si deux resolvers retournent chacun une valeur ? Variante : version « ciblée » avec `#[AsTargetedValueResolver]` + `#[ValueResolver]`.

## 7. Exceptions : de l'exception à la page d'erreur
Crée une exception métier `OutOfStockException` et un listener `kernel.exception` qui la convertit en réponse 409 JSON.
- Tests : statut 409 et corps JSON ; une autre exception garde le comportement par défaut.
- Réflexion : priorité du listener par rapport à `ErrorListener` (0 et -128) ? Quand faut-il `allowCustomResponseCode()` ? Compare avec l'implémentation de `HttpExceptionInterface` ou l'attribut `#[WithHttpStatus]` (voir `vendor/symfony/http-kernel/Attribute/`).

## 8. Sous-requêtes et `kernel.terminate`
(a) Fais un controller qui appelle `$this->forward(...)` ; dans un listener, journalise `isMainRequest()` pour chaque événement `kernel.request` et vérifie le nombre d'appels avec un logger de test.
(b) Ajoute un listener `kernel.terminate` qui enregistre dans un service « collecteur » ; teste via `KernelBrowser` que `terminate` est bien appelé et que la réponse est disponible dans `TerminateEvent::getResponse()`.
- Questions : `terminate` est-il appelé par `handle()` ? Qui l'appelle en production ? Que change `fastcgi_finish_request` pour l'utilisateur ? Pourquoi le mode debug le désactive (voir `HttpKernelRunner`) ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.


## Critères complémentaires des exercices 1–8

- **1–2** : conserver version/environnement avec la trace ; employer une URL volontairement inexistante, pas `/` dont le comportement peut évoluer. Prouver 404 avec capture et exception sans capture.
- **3–4** : tester réponse 200, erreur 404 et sous-requête ; retirer le flag de maintenance dans le nettoyage même si le test échoue. Un fichier témoin de maintenance n'est pas un mécanisme de verrouillage.
- **5–6** : tester explicitement un retour `Response` (aucune conversion par `kernel.view`) et un argument non concerné par le resolver (aucune modification).
- **7–8** : vérifier type de contenu et charge utile de l'erreur, puis absence de fuite vers une autre exception ; relever les appels séparément pour requête principale et sous-requête. Ne pas compter `terminate` comme une étape interne à `handle()`.

## 9. Journal du cycle normal (fondation)

Ajoute un collecteur de traces injecté dans des listeners à attributs. Chaque entrée porte événement, chemin et indicateur principal.
- Attendu pour une réponse normale : `request`, `controller`, `controller_arguments`, `response`, `finish_request` dans cet ordre, aucun `view`. Un appel direct à `handle()` n'ajoute pas `terminate` ; un appel explicite à `terminate()` l'ajoute ensuite.
- Preuve : un test de Kernel et une assertion sur la liste complète de ton collecteur, pas sur tous les listeners internes du framework.
- Indice : une priorité de listener ordonne un événement, pas l'ensemble du cycle.

## 10. Court-circuit et propagation (application)

Deux listeners de `kernel.request` de priorités 100 et 50 observent une route réservée. Le premier fournit une réponse 202 uniquement pour cette route.
- Attendu : la seconde trace de `request` est absente, contrôleur jamais appelé, `response` et `finish_request` présents ; une autre route exécute normalement les deux listeners.
- Indice : regarder `RequestEvent::setResponse()` et distinguer propagation de l'événement et poursuite du Kernel.

## 11. Sous-requête qui échoue (diagnostic)

Un contrôleur parent déclenche une sous-requête dont le contrôleur lève une exception connue. Traite cette exception en réponse 409 dans ton listener.
- Attendu pendant le fragment : current = fragment, parent = principale, main = principale ; après son retour, current = principale ; après l'appel complet sur une pile initialement vide, current/main/parent = null.
- Variante : appel direct avec `catch: false` et exception ordinaire ; l'exception remonte, la pile revient à son état initial et `finish_request` est observé.
- Indice : lire le `finally` de `HttpKernel::handle()` ; ne pas dépiler toi-même pour faire passer le test.

## 12. Deux requêtes, un service partagé (approfondissement)

Ajoute un service qui lit le chemin courant via `RequestStack` au moment de chaque appel. Utilise le même Kernel pour traiter `/premier` puis `/second`.
- Attendu : chaque réponse contient son propre chemin, aucun fragment ni chemin précédent ne persiste. Appeler le service hors requête suit un contrat explicite (par exemple retourner `null`).
- Indice : capturer la requête dans le constructeur est différent de capturer la pile.

## Défi final — incident reproductible

Compose maintenance, réponse normale, fragment et exception métier dans quatre routes de diagnostic. Pour chaque route, fournis statut, traces ordonnées, nombre d'appels de contrôleur et état final de la pile. Attendu : 503 en maintenance sans contrôleur, 200 normal, fragment identifié comme secondaire, exception métier convertie en 409, aucune contamination après une cinquième requête normale. Joins les commandes de diagnostic et explique où interviendrait `terminate`, sans ajouter de tâche asynchrone réelle.
