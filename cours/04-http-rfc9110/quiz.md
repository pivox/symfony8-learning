# Quiz — Module 04 : HTTP (RFC 9110)

Une seule bonne réponse sauf mention *(plusieurs réponses)*. Réponses en bas.

---

**Q1.** Quelles méthodes sont **sûres** au sens de la RFC 9110 ? *(plusieurs réponses)*
- A. GET
- B. HEAD
- C. PUT
- D. OPTIONS
- E. DELETE

**Q2.** Quelle affirmation est vraie ?
- A. Toute méthode idempotente est sûre
- B. Toute méthode sûre est idempotente
- C. POST est idempotent
- D. PUT n'est pas idempotent

**Q3.** Que retourne `Request::create('/', 'DELETE')->isMethodSafe()` et `->isMethodIdempotent()` ?
- A. `true` / `true`
- B. `false` / `true`
- C. `true` / `false`
- D. `false` / `false`

**Q4.** Un client envoie `PUT /articles/1` sur une route déclarée `methods: ['GET', 'POST']`. Quelle est la réponse ?
- A. 404
- B. 400
- C. 405 avec un header `Allow`
- D. 501

**Q5.** Quel code indique « authentifié mais non autorisé » ?
- A. 401
- B. 403
- C. 404
- D. 405

**Q6.** Quel code de redirection **conserve la méthode et le corps** et est permanent ?
- A. 301
- B. 302
- C. 307
- D. 308

**Q7.** Après un `POST` de formulaire réussi, quel code est classique pour le pattern Post/Redirect/Get ?
- A. 200
- B. 303 (ou 302)
- C. 304
- D. 204

**Q8.** Un client envoie un corps `Content-Type: text/plain` à un endpoint avec `#[MapRequestPayload]` (format non géré). Quel code est levé par Symfony ?
- A. 400
- B. 406
- C. 415
- D. 422

**Q9.** Quel code par défaut une violation de validation sur un DTO `#[MapRequestPayload]` produit-elle ?
- A. 400
- B. 404
- C. 422
- D. 500

**Q10.** Quel code par défaut un échec de mapping/validation d'un `#[MapQueryString]` produit-il ?
- A. 400
- B. 404
- C. 422
- D. 500

**Q11.** Quelle différence entre 406 et 415 ?
- A. 406 : la requête envoie un `Content-Type` non supporté ; 415 : aucune représentation ne satisfait `Accept`
- B. 406 : aucune représentation ne satisfait `Accept` ; 415 : le `Content-Type` du corps de la requête n'est pas supporté
- C. Aucune, ce sont des synonymes
- D. 406 concerne les cookies

**Q12.** Quelle méthode de `Request` retourne le format préféré en combinant `_format` et `Accept` ?
- A. `getRequestFormat()`
- B. `getContentTypeFormat()`
- C. `getPreferredFormat()`
- D. `getMimeType()`

**Q13.** Pourquoi envoyer `Vary: Accept` quand on utilise `getPreferredFormat()` ?
- A. Pour activer la compression
- B. Pour que les caches intermédiaires distinguent les représentations selon l'en-tête `Accept`
- C. Pour forcer HTTPS
- D. C'est obligatoire pour tout `Response`

**Q14.** Quelles sont les valeurs par défaut de `Cookie::create()` / `new Cookie()` dans Symfony 8 ? *(plusieurs réponses)*
- A. `httpOnly = true`
- B. `sameSite = 'lax'`
- C. `secure = null` (automatique selon HTTPS)
- D. `path = '/'`
- E. `sameSite = 'strict'`

**Q15.** Que signifie `SameSite=None` ?
- A. Le cookie n'est jamais envoyé cross-site
- B. Le cookie est envoyé aussi sur les requêtes cross-site ; les navigateurs exigent `Secure`
- C. Le cookie est supprimé à la fermeture du navigateur
- D. Le cookie est inaccessible à JavaScript

**Q16.** Comment supprimer un cookie côté serveur avec HttpFoundation ?
- A. `$response->headers->removeCookie('x')`
- B. `$response->headers->clearCookie('x', '/', null, ...)` (même nom, path, domaine)
- C. `unset($request->cookies['x'])`
- D. `$response->setStatusCode(205)`

**Q17.** Depuis Symfony 8.0, que se passe-t-il avec `_method=GET` sur un POST (override de méthode activé) ?
- A. La requête est traitée comme un GET
- B. L'override est ignoré : la méthode reste POST
- C. Une exception 500 est levée
- D. La requête est rejetée en 405

---


## Questions complémentaires — mise en situation

Une seule bonne réponse par question.

**Q18.** getFlashBag()->get('notice') est appelé deux fois sans nouvel ajout. Résultat ?
- A. Même liste deux fois
- B. Liste puis liste vide
- C. Exception au second appel
- D. Cookie supprimé

**Q19.** Que garantit HEAD concernant le contrôleur et le corps ?
- A. Contrôleur jamais exécuté
- B. Corps jamais envoyé, sans garantir l’absence de calcul
- C. Réponse toujours 204
- D. Même corps que GET

**Q20.** Un upload annonce image/png et un nom innocent. Quelle donnée suffit à garantir sa validité ?
- A. Le MIME annoncé
- B. L’extension seule
- C. Le nom d’origine seul
- D. Aucune : validation et stockage contrôlé restent nécessaires

**Q21.** Pour lire les octets d’une StreamedResponse, que faut-il considérer ?
- A. getContent contient toujours le résultat final
- B. Le callback exécuté lors de l’envoi
- C. Son statut doit être 206
- D. Elle exige Doctrine

**Q22.** L’application offre JSON/HTML ; Accept demande seulement application/xml. Que conclure ?
- A. Le framework garantit automatiquement un 406 dans tous les cas
- B. Il faut vérifier les formats offerts et produire le contrat 406 choisi
- C. Content-Type de la requête doit être changé
- D. Vary doit être supprimé

## Réponses

**Q1 — A, B, D.** RFC 9110 : GET, HEAD, OPTIONS, TRACE. Symfony prend aussi en charge QUERY comme extension sûre. PUT et DELETE sont idempotents, pas sûrs.

**Q2 — B.** Sûre ⇒ idempotente, pas l'inverse. POST n'est pas idempotent ; PUT l'est.

**Q3 — B.** `isMethodSafe()` : `GET, HEAD, OPTIONS, TRACE, QUERY` ; `isMethodIdempotent()` inclut `DELETE`.

**Q4 — C.** `MethodNotAllowedHttpException` (405) avec `Allow: GET, POST`.

**Q5 — B.** 401 = non authentifié (avec `WWW-Authenticate`) ; 403 = refus.

**Q6 — D.** 308 : permanent, méthode conservée. 301/302 peuvent transformer POST en GET ; 307 conserve la méthode mais est temporaire.

**Q7 — B.** 303 « See Other » force un GET sur la cible (302 est souvent utilisé aussi, par défaut `redirectToRoute()`).

**Q8 — C.** `RequestPayloadValueResolver` : `UnsupportedMediaTypeHttpException` (415) si le format du Content-Type est inconnu ou n'est pas dans `acceptFormat`.

**Q9 — C.** `validationFailedStatusCode` vaut `Response::HTTP_UNPROCESSABLE_ENTITY` (422) par défaut dans `MapRequestPayload`.

**Q10 — B.** `MapQueryString::$validationFailedStatusCode` vaut `HTTP_NOT_FOUND` (404) par défaut ; idem pour `MapQueryParameter`.

**Q11 — B.** 406 concerne la réponse (`Accept*`), 415 le corps de la requête (`Content-Type`).

**Q12 — C.** `getPreferredFormat()` regarde le format de requête (`_format`) puis `Accept`. `getRequestFormat()` ne regarde pas `Accept`.

**Q13 — B.** Sans `Vary: Accept`, un cache peut servir du JSON à un client qui veut du HTML (docblock de `getPreferredFormat()`).

**Q14 — A, B, C, D.** Valeurs lues dans `Cookie::__construct()` : `path '/'`, `secure null`, `httpOnly true`, `sameSite 'lax'`, `expire 0`.

**Q15 — B.** `None` = envoyé en cross-site ; les navigateurs modernes exigent `Secure` avec `None`.

**Q16 — B.** `ResponseHeaderBag::clearCookie($name, $path, $domain, $secure, $httpOnly, $sameSite, $partitioned)` envoie un cookie expiré ; le path/domaine doivent correspondre.

**Q17 — B.** HttpFoundation 8.0 : « Drop HTTP method override support for methods GET, HEAD, CONNECT and TRACE » ; `Request::getMethod()` retourne la méthode d'origine dans ce cas. La config `allowed_http_method_override` rejette aussi ces valeurs.

**Q18 — B.** get consomme les messages de ce type. A correspond à peek, C n’est pas le contrat, D confond le sac de flashes avec le cookie de session.

**Q19 — B.** L’absence de corps est une propriété de la réponse. A invente une optimisation, C impose un statut sans raison, D viole HEAD.

**Q20 — D.** Les métadonnées client sont déclaratives. A, B et C peuvent être falsifiés et ne prouvent ni contenu acceptable ni chemin sûr.

**Q21 — B.** La production du corps est différée au callback. A transpose une Response classique, C confond streaming et plage HTTP, D ajoute une dépendance sans rapport.

**Q22 — B.** Un helper de préférence ne connaît pas nécessairement l’offre métier. A attribue une garantie trop large au helper, C confond corps entrant et sortie, D détruit la distinction de variantes.

