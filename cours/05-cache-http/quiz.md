# Quiz — Module 05 : Cache HTTP

Réponds sans regarder la doc. Une ou plusieurs bonnes réponses : c'est indiqué entre parenthèses. Réponses en bas.

**Q1 (une réponse).** Quelle directive définit une durée de fraîcheur propre aux caches partagés ?
A. `max-age` B. `s-maxage` C. `private` D. `immutable`

**Q2 (une réponse).** Que signifie `Cache-Control: no-cache` ?
A. La réponse ne doit pas être stockée B. La réponse peut être stockée mais doit être revalidée avant réutilisation C. La réponse est stockée uniquement côté serveur D. Désactive `Vary`

**Q3 (une réponse).** Quel code HTTP renvoie `Response::isNotModified()` + `return $response` lorsque l'ETag correspond ?
A. 200 B. 204 C. 304 D. 412

**Q4 (plusieurs réponses).** Quels en-têtes de requête sont utilisés par `Response::isNotModified()` pour produire éventuellement un 304 sur un `GET` ?
A. `If-None-Match` B. `If-Modified-Since` C. `If-Match` D. `Accept-Language`

**Q5 (une réponse).** Dans Symfony 8.1, où se trouve `#[Cache]` ?
A. `Sensio\Bundle\FrameworkExtraBundle\Configuration\Cache` B. `Symfony\Component\HttpKernel\Attribute\Cache` C. `Symfony\Component\HttpFoundation\Cache` D. `Symfony\Bundle\FrameworkBundle\Attribute\Cache`

**Q6 (plusieurs réponses).** Quels paramètres le constructeur de `#[Cache]` accepte-t-il ?
A. `maxage` B. `smaxage` C. `etag` D. `ttl`

**Q7 (une réponse).** `Vary: Accept-Language` signifie :
A. Le client doit envoyer la langue B. Le cache doit stocker une variante par valeur de `Accept-Language` C. La réponse est privée D. Le contenu est compressé

**Q8 (une réponse).** Quelle est la valeur par défaut de `private_headers` du `HttpCache` ?
A. `[]` B. `['Cookie']` C. `['Authorization', 'Cookie']` D. `['Set-Cookie']`

**Q9 — enrichissement ESI hors certification (une réponse).** Que se passe-t-il avec `render_esi()` quand le reverse proxy ne supporte pas ESI ?
A. Erreur 500 B. Le fragment est rendu inline (sous-requête interne) C. Le fragment est omis D. Le fragment est rendu côté client en JS

**Q10 — enrichissement ESI hors certification (plusieurs réponses).** Quelles conditions sont nécessaires pour utiliser ESI avec Symfony ?
A. `framework.esi: true` B. Un reverse proxy envoyant `Surrogate-Capability` C. `framework.fragments` activé pour la route de fragments D. Twig doit être désactivé

**Q11 (une réponse).** Une page publique démarre la session (écriture de flash message). Résultat sur `Cache-Control` ?
A. Inchangé B. `private, max-age=0` ajouté automatiquement C. `no-store` D. `public` forcé

**Q12 — enrichissement ESI hors certification (une réponse).** Deux fragments : page `max-age=300`, fragment ESI `max-age=10`. Fraîcheur de la page assemblée par le cache ?
A. 300 B. 10 C. 310 D. Non déterminé

**Q13 (une réponse).** Quel statut n'est PAS cachable par défaut selon `Response::isCacheable()` ?
A. 200 B. 301 C. 404 D. 500

**Q14 (une réponse).** `Response::setSharedMaxAge(3600)` produit :
A. `max-age=3600` B. `s-maxage=3600` C. `public` seulement D. `Expires` seulement

**Q15 (plusieurs réponses).** Pour qu'un 304 fasse réellement économiser du travail au serveur, il faut :
A. Appeler `isNotModified()` avant de construire le contenu B. Utiliser `#[Cache(etag: ..., lastModified: ...)]` pour valider avant le contrôleur C. Appeler `isNotModified()` après le rendu D. Ajouter `Vary: *`

---


## Questions complémentaires — mise en situation

Une seule bonne réponse par question.

**Q16.** max-age=60, s-maxage=600, Age=45 : quel budget de fraîcheur reste au cache partagé (sans délai additionnel) ?
- A. 15 secondes
- B. 555 secondes
- C. 645 secondes
- D. 600 secondes

**Q17.** La réponse expose ETag et Last-Modified. ETag client différent, date identique : résultat de isNotModified() ?
- A. true, 304
- B. false, réponse normale
- C. Toujours 412
- D. Toujours exception

**Q18.** Deux appels KernelBrowser donnent Cache-Control: public. Ont-ils prouvé un hit proxy ?
- A. Oui, public signifie hit
- B. Non, il faut passer par un proxy et observer le store ou le compteur d’origine
- C. Oui si la seconde réponse est 200
- D. Seulement si le JSON est identique

**Q19.** Quel couple protège une réponse personnelle dont aucun stockage n’est souhaité ?
- A. public et Vary: Cookie
- B. private et no-store
- C. s-maxage=3600 uniquement
- D. ETag uniquement

**Q20.** Que contrôle Vary par rapport à ETag ?
- A. Vary choisit les variantes, ETag valide la représentation choisie
- B. Ce sont deux noms pour le même en-tête
- C. ETag choisit automatiquement la langue
- D. Vary empêche toujours le stockage

## Réponses

1. **B**. `s-maxage` définit une durée propre au cache partagé ; `private` lui interdit le stockage, tandis que max-age et immutable ne sont pas propres au partagé.
2. **B**. `no-store` interdit le stockage ; `no-cache` impose la revalidation.
3. **C**. `setNotModified()` met 304 et vide le corps.
4. **A, B**. `If-Match` sert à la concurrence optimiste (412) ; `Accept-Language` n'est pas conditionnel.
5. **B**. Intégré à HttpKernel depuis 6.2 (plus de sensio).
6. **A, B, C**. `ttl` n'existe pas (`Cache` expose `expires, maxage, smaxage, public, mustRevalidate, vary, lastModified, etag, maxStale, staleWhileRevalidate, staleIfError, noStore, if`).
7. **B**. Une entrée de cache par valeur de l'en-tête listé.
8. **C**. Vérifié dans `HttpCache::__construct`.
9. **B**. `InlineFragmentRenderer` : rendu interne, sans bénéfice de cache.
10. **A, B, C**. Twig est nécessaire pour `render_esi` (D est faux).
11. **B**. Dans ce cas explicitement public avec écriture de flash, le listener rend la réponse privée, met max-age à 0 et ajoute must-revalidate. Ne pas généraliser max-age=0 à toute réponse déjà privée.
12. **B**. Pour ces réponses fraîches sans Age initial ni autre directive restrictive, la réponse assemblée retient 10 s ; les ressources source et fragment gardent leur stockage distinct dans le proxy.
13. **D**. Liste : 200, 203, 300, 301, 302, 404, 410 (et il faut un validateur ou une fraîcheur).
14. **B**.
15. **A, B**. En C le travail est déjà fait ; `Vary: *` détruit le cache.

**Q16 — B.** Le partagé prend 600 moins 45. A applique la durée navigateur, C additionne un âge consommé, D ignore cet âge.

**Q17 — B.** La comparaison ETag prend le dessus dans ce cas. A ferait un ou erroné avec la date, C concerne un autre type de précondition, D n’est pas le contrat de cette méthode.

**Q18 — B.** L’en-tête annonce une politique, pas l’exécution réelle d’un cache. A confond éligibilité et preuve ; C et D sont compatibles avec deux calculs complets.

**Q19 — B.** Private exclut le partagé et no-store interdit le stockage. A peut rendre partageables des variantes et n’autorise personne ; C accorde une fraîcheur partagée ; D valide une représentation sans contrôler son stockage.

**Q20 — A.** Ils interviennent à des étapes différentes. B les confond, C prête une négociation automatique au validateur, D est faux pour un Vary ordinaire comme Accept-Language.

