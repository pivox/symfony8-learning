# Quiz — Module 15 : HttpClient

[Cours](README.md) · [Exercices](exercices.md)

Questions d'entraînement originales, socle Symfony 8.0. Une seule réponse par question sauf mention contraire.

**Q1 (une réponse).** Quel type injecter pour pouvoir substituer le client ?
A. `CurlHttpClient` exclusivement B. `HttpClientInterface` des Contracts C. `Request` D. `Response`

**Q2 (une réponse).** Où une panne réseau peut-elle se manifester ?
A. Seulement dans `request()` B. Seulement au démarrage du Kernel C. Pendant la lecture de la réponse aussi D. Jamais si le statut est 200

**Q3 (une réponse).** `toArray(false)` garantit :
A. Aucune exception B. Un tableau même pour du HTML C. Aucune erreur réseau D. Pas d'exception déclenchée uniquement par le statut HTTP

**Q4 (une réponse).** Quel paramètre exprime un corps JSON ?
A. `query` B. `json` C. `base_uri` D. `user_data`

**Q5 (une réponse).** `timeout` limite principalement :
A. L'inactivité réseau B. La durée totale dans tous les cas C. Le nombre d'appels D. La taille du corps

**Q6 (une réponse).** Quelle option borne toute la requête/réponse ?
A. `max_redirects` B. `buffer` C. `max_duration` D. `query`

**Q7 (une réponse).** `withOptions()` :
A. Modifie obligatoirement l'original B. Envoie une requête C. Vide les réponses D. Retourne un nouveau client

**Q8 (une réponse).** Quel double évite le réseau ?
A. `MockHttpClient` B. Un vrai client avec timeout court C. Une URL improbable D. Un client sans TLS

**Q9 (une réponse).** La factory de mock reçoit :
A. Seulement le statut B. Le contrôleur C. Méthode, URL, options D. La session de l'utilisateur

**Q10 (une réponse).** Pourquoi lancer plusieurs requêtes avant de lire leurs réponses ?
A. Pour garantir leur ordre d'arrivée B. Pour permettre leur exécution concurrente C. Pour supprimer les exceptions D. Pour les rendre idempotentes

**Q11 (une réponse).** Deux reprises après l'appel initial donnent au maximum :
A. Une tentative B. Deux C. Quatre D. Trois

**Q12 (une réponse).** Répéter un POST de réservation nécessite d'abord :
A. Un contrat de déduplication ou une garantie d'idempotence adaptée B. Un `sleep` C. Un cache navigateur D. Une URL absolue

**Q13 (une réponse).** Un JSON décodé avec succès :
A. Est authentifié B. A nécessairement tous les champs C. Doit encore respecter le contrat métier attendu D. Est toujours un produit valide

**Q14 (une réponse).** Base `https://catalogue.example/v1/`, chemin `/products` :
A. Donne `/v1/products` B. Donne `/products` C. Est forcément invalide D. Ajoute un paramètre `v1`

**Q15 (plusieurs réponses).** Que prouve un test avec `MockHttpClient` ?
A. La disponibilité réelle du fournisseur B. Son temps de réponse réel C. Le traitement d'une réponse simulée D. Les paramètres envoyés par le service si le test les vérifie

## Réponses

1. **B** — Le contrat accepte le client réel et le double. A impose une implémentation ; C et D ne sont pas des clients.
2. **C** — La réponse est différée. A et B limitent abusivement le moment d'échec ; D ignore les pannes pendant la lecture du corps.
3. **D** — `false` ne concerne que les statuts. Le transport et le décodage peuvent encore échouer : A, B et C sont faux.
4. **B** — `json` encode le corps. `query` agit sur l'URL, `base_uri` sur sa résolution et `user_data` transporte des métadonnées locales.
5. **A** — Un flux lent mais actif peut dépasser la durée souhaitée. B confond avec `max_duration` ; C et D ne sont pas des délais.
6. **C** — Les autres options concernent redirections, stockage du corps et paramètres d'URL.
7. **D** — L'original reste utilisable avec ses options. Il n'y a ni envoi ni vidage implicite.
8. **A** — B et C tentent toujours du réseau ; D le rend moins sûr sans le supprimer.
9. **C** — Ces informations permettent de vérifier le contrat HTTP. Le mock ne reçoit ni contrôleur ni session automatiquement.
10. **B** — L'ordre d'arrivée n'est pas garanti ; concurrence, erreurs et idempotence sont des questions distinctes.
11. **D** — Une tentative initiale plus deux reprises. Les autres comptes omettent ou ajoutent des tentatives.
12. **A** — Réessayer une écriture peut doubler son effet ; attente, cache et forme d'URL ne règlent pas cette propriété métier.
13. **C** — Le décodage vérifie une représentation, pas l'identité ni la présence et le type des champs attendus.
14. **B** — Le slash initial repart à la racine. Sans slash, le chemin relatif conserverait `/v1/` ; il ne s'agit pas d'une query string.
15. **C, D** — Le test vérifie ton intégration simulée. Disponibilité et performance réelles exigent d'autres observations.
