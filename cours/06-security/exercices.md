# Exercices — Module 06

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Code dans `src/Security/` (namespace `App\Security`), tests dans `tests/Security/` (namespace `App\Tests\Security`) : `php bin/phpunit tests/Security`. Écris le test d'abord. Les voters se testent en **unitaire** (pas besoin du kernel) ; les firewalls/`access_control` en `WebTestCase` avec `$client->loginUser()`.

Pas de Doctrine installé : utilise un provider `memory` ou un provider custom en mémoire.

## 1. User et provider
Crée `App\Security\User` (implémente `UserInterface` + `PasswordAuthenticatedUserInterface`) et configure un provider `memory` ou un `UserProviderInterface` custom (`InMemoryUserProvider` maison).
- Tests : `loadUserByIdentifier` pour un identifiant inconnu lève `UserNotFoundException` ; `refreshUser` d'une autre classe lève `UnsupportedUserException`.
- Réflexion : pourquoi `eraseCredentials()` n'existe-t-il plus ? Qu'est-ce qui se passe à chaque requête avec `refreshUser` ?

## 2. `access_control` et firewalls
Configure : `dev` (`security: false`), `main`. Règles : `^/admin` ⇒ `ROLE_ADMIN`, `^/profil` ⇒ `ROLE_USER`. Crée deux contrôleurs minimalistes.
- Tests fonctionnels : anonyme ⇒ redirection/401 ; `ROLE_USER` sur `/admin` ⇒ 403 ; admin ⇒ 200.
- Réflexion : inverse l'ordre de deux règles qui se chevauchent : que change-t-il ? Pourquoi `security: false` sur le firewall `dev` est une bonne idée... et un danger pour d'autres chemins ?

## 3. Authenticator custom par token d'API
Écris `ApiTokenAuthenticator` (`AbstractAuthenticator`) lisant `X-API-TOKEN` : renvoie `SelfValidatingPassport` + `UserBadge`. Firewall `api` `stateless: true`.
- Tests : pas d'en-tête ⇒ `supports()` false (requête non traitée) ; token invalide ⇒ 401 JSON (échec) ; token valide ⇒ 200.
- Réflexion : que veut dire retourner `null` de `supports()` vs `false` ? Pourquoi `onAuthenticationSuccess` retourne-t-il `null` ici ?

## 4. Login JSON avec Passport complet
Écris un authenticator de login (email + mot de passe) avec `Passport`, `UserBadge`, `PasswordCredentials` et `PasswordUpgradeBadge` (ou `RememberMeBadge`).
- Tests : bons identifiants ⇒ succès ; mauvais mot de passe ⇒ `BadCredentialsException`/401 ; utilisateur inconnu ⇒ **même** réponse qu'un mauvais mot de passe.
- Réflexion : quel écouteur vérifie le mot de passe ? Que se passe-t-il si tu oublies d'ajouter `PasswordCredentials` (ou un badge requis) : essaie et lis l'erreur.

## 5. Écouteurs d'événements de sécurité
Crée un subscriber qui (a) sur `LoginSuccessEvent` enregistre dans un service `LoginAudit` l'identifiant et le firewall ; (b) sur `LoginFailureEvent` incrémente un compteur ; (c) sur `LogoutEvent` modifie la réponse de redirection.
- Tests : après une requête de login réussie/échouée, `LoginAudit` contient la trace attendue.
- Observe : `bin/console debug:event-dispatcher` sur ces événements ; note les priorités des écouteurs du core.
- Réflexion : pourquoi `CheckPassportEvent` n'est-il pas un bon endroit pour journaliser un succès ?

## 6. Voter
Crée `PostVoter` (attributs `POST_VIEW`, `POST_EDIT`) sur un `Post` (id, auteur, publié). Règles : lecture si publié ou si auteur ; édition seulement si auteur ; un `ROLE_ADMIN` peut tout.
- Tests unitaires : une ligne par cas, y compris attribut non supporté ⇒ `ACCESS_ABSTAIN`, sujet d'un autre type ⇒ abstention, utilisateur anonyme ⇒ refus sur édition et lecture privée (lecture publique autorisée).
- Utilise `Vote::addReason()` pour expliquer un refus ; vérifie-le dans le test.
- Réflexion : pourquoi `supports()` doit-il être strict ? Quel est l'effet d'un voter qui renvoie toujours `true` en stratégie `affirmative` ? et en `unanimous` ?

## 7. Stratégies d'AccessDecision
Sans kernel, construis un `AccessDecisionManager` avec trois voters factices (anonymes : un grant, un deny, un abstain) et teste les quatre stratégies (`AffirmativeStrategy`, `ConsensusStrategy`, `UnanimousStrategy`, `PriorityStrategy`) + options `allowIfAllAbstain` / `allowIfEqualGrantedDenied`.
- Tests : tableau de vérité (jeu de voters × stratégie) avec un `#[DataProvider]`.
- Puis dans l'app : change `security.access_decision_manager.strategy` en `unanimous` et observe l'impact sur ton `PostVoter` / `RoleVoter` via les tests et `debug:event-dispatcher` (ou le panneau Security après installation facultative du profiler).
- Réflexion : pour la stratégie `priority`, comment définir l'ordre des voters ?

## 8. `#[IsGranted]` et hiérarchie de rôles
Utilise `#[IsGranted('POST_EDIT', subject: 'post', statusCode: 404)]` sur un contrôleur et `#[IsGranted('ROLE_ADMIN')]` sur une classe. Ajoute `role_hierarchy: { ROLE_ADMIN: [ROLE_USER] }`.
- Tests : auteur ⇒ 200 ; autre utilisateur ⇒ 404 (et non 403) ; admin hérite de `ROLE_USER`.
- Réflexion : où s'exécute `#[IsGranted]` dans le cycle `kernel.*` (cf. module 02) ? Pourquoi `getRoles()` ne retourne-t-il pas `ROLE_USER` pour un admin ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.

## Critères complémentaires pour les exercices 1–8

Appliquer la [méthode commune](../README.md). Les scénarios d'authentification utilisent des requêtes réelles ; réserver `loginUser()` aux scénarios d'autorisation. Une assertion porte sur le statut **et** sur l'absence ou la présence d'un effet métier.

| Exercice | Acceptation supplémentaire |
|---|---|
| 1 | Identifiant connu → même identifiant et rôles attendus ; inconnu → exception précise, jamais `null`. Le provider `memory` natif produit un `InMemoryUser` : utiliser le provider custom si l'exercice exige la classe `App\Security\User`. |
| 2 | Choisir explicitement 302 pour le site et 401 pour l'API ; vérifier `Location` pour 302, et aucun contenu admin sur les refus. |
| 3 | Sur une route API protégée, aucun token → 401 via entry point même si `supports()` est faux ; deux requêtes stateless exigent deux en-têtes. |
| 4 | Identifiant inconnu et mauvais mot de passe → même statut et même message public ; aucun mot de passe ni hash dans le JSON. Préférer `json_login` natif pour le JSON, ou sa sous-classe adaptée ; ne pas inventer un parseur JSON manuel. |
| 5 | Exactement une trace par événement attendu ; aucun succès après credentials invalides. Consulter le dispatcher du firewall. |
| 6 | Cas publié+anonyme et privé+anonyme doivent être spécifiés séparément : ici autoriser la lecture publique, refuser l'édition et la lecture privée. Le refus anonyme demandé plus haut vise ces actions protégées. |
| 7 | Grant+deny+abstain : affirmative=true, consensus=true par défaut, unanimous=false, priority=vote du premier non-abstention ; tous abstain=false. |
| 8 | Séparer le contrôleur réservé à l'admin de la route d'édition de l'auteur : sinon le `ROLE_ADMIN` de classe empêche le scénario auteur. Vérifier 404 pour le non-auteur et absence d'effet. |

## 9. Parcours guidé — authentification en session

Configurer un login de formulaire et deux utilisateurs en mémoire (lecteur et éditeur), sans Doctrine. Définir une route `/lesson/profile` accessible à `ROLE_USER` et `/lesson/publish` réservée à `ROLE_EDITOR`.

- Acceptation : anonyme sur profil → 302 vers la connexion ; mauvais mot de passe → échec sans accès au profil ; bon mot de passe → redirection puis profil 200 avec le même client ; lecteur sur publication → 403 ; éditeur → 200.
- Vérifier qu'une requête sans credentials après le login réussit via la session, tandis qu'un nouveau client reste anonyme.
- Indice : tracer séparément la réponse de connexion et celle de la requête suivante ; ne pas utiliser `loginUser()` ici.

## 10. Parcours intermédiaire — CSRF et déconnexion

Ajouter la protection CSRF à la connexion et à la déconnexion. Pour isoler la variante avec session, choisir les identifiants `lesson_login` et `lesson_logout`, absents de la liste sans état de la recette.

- Acceptation : token de connexion absent ou altéré → authentification refusée ; token correct + credentials corrects → profil 200. Logout avec token valide → redirection, puis profil redirigé vers la connexion avec le même client.
- Une tentative de logout avec token invalide doit échouer et le profil doit rester accessible ensuite. Tester l'effet, pas seulement le message d'erreur.
- Comparaison : revenir à l'identifiant sans état `authenticate` et tester les en-têtes d'origine explicitement ; expliquer pourquoi ce deuxième mécanisme ne se résume pas au même test de chaîne de token.
- Indice : observer la configuration résolue CSRF et les listeners `LogoutEvent` avant d'interpréter les résultats.

## 11. Parcours avancé — refresh et perte des droits

Avec un provider custom de test, préparer deux versions d'un utilisateur : avant et après retrait de `ROLE_EDITOR`. Tester le refresh entre deux requêtes. Le stockage de test doit être contrôlable entre requêtes malgré le redémarrage du kernel.

- Acceptation : la première publication réussit ; après changement des rôles, la suivante ne publie rien. Avec la comparaison standard des tokens, le changement de rôles peut provoquer une déconnexion : relever la redirection de l'entry point, sans supposer systématiquement 403.
- Prouver séparément qu'une nouvelle authentification comme lecteur donne 403 ; comparer identité perdue et identité conservée sans permission.
- Indice : `ContextListener::hasUserChanged()` et `EquatableInterface`, sans contourner la comparaison pour faire passer le test.

## Défi final — espace de publication protégé

Assembler connexion, profil, publication avec voter d'auteur et logout. Prévoir un article public et un privé, auteur et lecteur distincts. Aucun stockage Doctrine requis : fixtures et dépendances de test suffisent.

Livrables : matrice HTTP (anonyme, mauvais credentials, lecteur, auteur, admin), tests du cycle réel connexion → action → sortie → refus, et une courte trace des événements. Acceptation : aucune publication refusée n'appelle le service métier, lecture publique autorisée, édition uniquement pour auteur/admin, CSRF invalide sans effet, aucun secret dans les réponses. Justifier les différences 302/401/403/404 sans fournir un statut interchangeable. Indice final : partir d'une route et d'un droit, puis élargir la matrice ; ne pas tout tester uniquement au niveau du voter.

## 12. Compte désactivé malgré un mot de passe correct

**Niveau : intermédiaire · Durée : 50–65 min · Prérequis : exercices 9–10 et [user checker](approfondissement.md).**

Dans le parcours de formulaire existant, ajouter un `UserCheckerInterface` sélectionné par le firewall. Deux utilisateurs fixture partagent le rôle `ROLE_USER` : `alice@example.test` active et `bruno@example.test` désactivé. Leur mot de passe de démonstration est `atelier-42`, hashé avec le service Symfony lors de la préparation des fixtures.

Le provider custom charge un champ `enabled` sans base de données. Chaque tentative utilise un client neuf, un vrai formulaire et, si activé, un token CSRF valide obtenu par ce client. Un collecteur de test distingue précontrôle, postcontrôle et succès final ; l'action protégée compte ses appels.

Critères de réussite :

- Alice avec le bon mot de passe : connexion réussie, puis profil 200 avec le même client ; précontrôle et postcontrôle observés avant `LoginSuccessEvent`.
- Bruno avec le bon mot de passe : refus par le checker ; aucun `LoginSuccessEvent`, aucun accès métier, puis profil redirigé vers la connexion avec le même client.
- Cas négatif : Alice avec `incorrect` échoue aussi, mais le postcontrôle et le succès final ne sont pas observés. Distinguer cet échec de celui d'état du compte.
- Pour le refus de Bruno, vérifier qu'une exception d'état de compte adaptée a été utilisée, sans exposer de détails administratifs dans la page publique.
- Le checker fonctionne avec la signature Symfony 8 de `checkPostAuth()` ; son seul enregistrement comme service ne suffit pas sans `user_checker` sur le firewall.

Livrable : trois traces, réponses après redirection et compteur d'effet. Prévoir un collecteur observable malgré le redémarrage du Kernel de BrowserKit, ou relever son état après chaque requête ; ne pas perdre la preuve entre les appels.

**Indice 1 :** les credentials et le statut administratif du compte sont deux vérifications séparées.

**Indice 2 :** comparer `UserCheckerListener` et les événements `AuthenticationSuccessEvent`/`LoginSuccessEvent` ; le premier peut encore déboucher sur un refus.

## 13. Remplacer le transport custom par `access_token`

**Niveau : avancé · Durée : 55–75 min · Prérequis : exercice 3 et [authenticator natif](approfondissement.md).**

Créer un firewall d'atelier limité à `^/lesson/token`, placé avant le firewall général, avec `stateless: true`, le mécanisme natif `access_token` et un handler applicatif. Utiliser seulement l'extracteur Bearer par défaut, un provider en mémoire et un entry point explicite renvoyant 401 avec challenge Bearer.

Données de démonstration, sans valeur réelle : `lesson-reader` identifie `reader@example.test` avec `ROLE_USER` ; `lesson-editor` identifie `editor@example.test` avec `ROLE_USER` et `ROLE_EDITOR`. Tout autre token, notamment `lesson-revoked`, est refusé. `/lesson/token/profile` exige `ROLE_USER`, `/lesson/token/publish` exige `ROLE_EDITOR`.

Critères de réussite par véritables requêtes HTTP, sans `loginUser()` :

- `Authorization: Bearer lesson-reader` sur le profil → 200 avec l'identifiant attendu ; `lesson-editor` sur publication → 200 et un effet métier.
- Token absent → 401 par l'entry point ; token inconnu ou révoqué → 401 par l'échec d'authentification. Les réponses portent un challenge `WWW-Authenticate` de schéma Bearer.
- Cas négatif d'autorisation : token lecteur sur publication → 403, sans effet métier ; il ne devient pas un échec de validation du token.
- Un appel valide suivi d'un appel sans header, avec le même client, redonne 401 : aucune authentification n'est récupérée de la session.
- Fournir seulement `?access_token=lesson-editor` ne donne aucun accès, puisque cet extracteur n'a pas été activé.
- Aucun token brut n'apparaît dans le corps des réponses ni dans les traces métier ; les fixtures ne servent qu'aux tests locaux.

Livrable : matrice HTTP, diagnostic du firewall sélectionné et comparaison des responsabilités avec l'authenticator custom de l'exercice 3. Le handler vérifie le token ; les rôles et voters décident toujours des actions permises.

**Indice 1 :** l'extraction HTTP est déjà fournie ; le contrat à implémenter côté application retourne un `UserBadge` après vérification.

**Indice 2 :** un authenticator non applicable ne fabrique pas automatiquement la réponse à l'anonyme refusé ; vérifier séparément entry point, handler d'échec et ordre des firewalls.
