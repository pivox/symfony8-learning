# Approfondissement — État du compte et authentification par access token

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Prérequis : Passport, provider, authenticator et cycle HTTP ; prévoir 40 minutes.
API du socle **Symfony 8.0**, vérifiées sur **8.1.8** ; aucune nouveauté 8.1 n'est requise ici.
SecurityBundle est installé ; les utilisateurs et tokens des ateliers sont des fixtures locales en mémoire.
Le firewall actuel n'active pas ces mécanismes : les exercices demandent leur configuration explicite.

## 1. Un mot de passe valide ne suffit pas pour un compte désactivé

La preuve d'identité et l'état administratif du compte sont deux conditions distinctes.
Un `UserCheckerInterface` exprime une restriction du compte : suspendu, désactivé ou expiré.
Un voter décide plutôt si l'utilisateur peut effectuer une action sur un objet précis.
Le checker ne doit donc pas devenir un second routeur d'autorisations métier.

```php
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\User\UserInterface;

// Signatures à implémenter dans un UserCheckerInterface :
// checkPreAuth(UserInterface $user): void
// checkPostAuth(UserInterface $user, ?TokenInterface $token = null): void
```

Le paramètre optionnel `$token` de `checkPostAuth()` fait partie du contrat Symfony 8.
Une ancienne implémentation recopiée sans cette signature doit être adaptée au contrat installé.
Déclarer le checker comme service autoconfiguré ne sélectionne pas automatiquement le firewall qui l'utilise.
L'option `user_checker` du firewall établit ce lien.

### Situer les contrôles dans les événements

Le précontrôle est déclenché sur `CheckPassportEvent`, lorsque l'utilisateur du Passport est résolu.
Le postcontrôle intervient sur `AuthenticationSuccessEvent`, avec le token créé.
Ce dernier événement interne précède le succès final de login : le checker peut encore refuser la tentative.
Une trace sur `LoginSuccessEvent` ne doit alors pas apparaître.

Pour refuser, lever une exception d'état de compte adaptée, par exemple `DisabledException`.
Une `RuntimeException` quelconque n'exprime pas un échec d'authentification exploitable de la même façon.
Les messages publics doivent rester ceux prévus par le contrat d'authentification, sans exposer un dossier administratif.
Une réponse de refus ne doit produire aucun effet métier ni session donnant ensuite accès au profil.

| Règle | Emplacement principal | Preuve attendue |
|---|---|---|
| Le mot de passe correspond | Credentials du Passport | Mauvais secret refusé |
| Le compte est utilisable | User checker | Bon secret mais compte désactivé refusé |
| Cet article appartient au lecteur | Voter | Identité reconnue mais opération interdite |
| Le compte a changé entre requêtes | Provider/refresh et politique de session | Scénario de requêtes successives |

Un checker n'est pas un hook « première connexion de la journée ».
Il peut s'exécuter lors de réauthentifications et de mécanismes comme l'impersonation pour le postcontrôle.
Éviter d'y envoyer un email ou d'y incrémenter un compteur de connexion réussi.
Le badge `PreAuthenticatedUserBadge` permet au framework de sauter le précontrôle dans certains mécanismes ; ne pas le confondre avec `SelfValidatingPassport`.
Ce dernier dispense d'une vérification de credentials supplémentaire, pas des règles d'état du compte.

## 2. Utiliser l'authenticator natif `access_token`

L'exercice de l'authenticator custom enseigne le contrat complet.
Quand le besoin est « extraire un token Bearer puis retrouver un utilisateur », Security possède déjà le mécanisme `access_token`.
Le code applicatif peut se concentrer sur `AccessTokenHandlerInterface`.

```php
use Symfony\Component\Security\Http\Authenticator\Passport\Badge\UserBadge;

// Contrat du handler :
// public function getUserBadgeFrom(string $accessToken): UserBadge;
```

Le handler vérifie le token, puis renvoie un badge permettant de charger l'utilisateur.
Un token inconnu, expiré ou révoqué doit échouer avant de créer une identité utilisable.
`BadCredentialsException` convient pour un refus générique ; ne pas retourner un utilisateur anonyme en simulant un succès.
Le handler natif ne décide pas des droits sur une ressource : les règles d'accès et voters continuent ensuite.

La configuration relie `access_token.token_handler` au service et choisit `stateless: true` pour ce scénario API.
Le transport par défaut est `Authorization: Bearer …` via `HeaderAccessTokenExtractor`.
Les extracteurs de query string et de corps ne sont pas activés implicitement pour l'atelier.
Cela rend testable le fait qu'un token placé dans `?access_token=…` ne donne aucun accès.
Ne jamais considérer la simple présence d'une chaîne Bearer comme une validation.

Le mécanisme utilise un `SelfValidatingPassport` parce que le handler a déjà effectué la vérification du token.
Un user checker du firewall peut toujours rejeter le compte correspondant.
Avec `stateless: true`, le prochain appel protégé doit à nouveau présenter son token.
Un premier succès ne crée pas un droit permanent dans les cookies du client.

| Requête sur une route protégée | Chemin du traitement |
|---|---|
| En-tête Bearer valide | Extraction → handler → utilisateur → contrôle d'accès |
| Token inconnu | Handler refuse → échec d'authentification |
| En-tête absent | Authenticator non applicable → entry point configuré |
| Token valide, rôle insuffisant | Authentification réussie → refus d'autorisation |
| Token valide, compte désactivé | Checker refuse → aucun appel métier |

Le handler d'échec natif répond notamment 401 avec `WWW-Authenticate: Bearer …`.
L'absence de credentials demande aussi un entry point adapté : ne pas supposer que ce handler d'échec traite automatiquement toutes les absences.
Configurer explicitement l'entry point de l'atelier pour un 401 cohérent.
Un corps JSON personnalisé est un choix de réponse supplémentaire, pas une propriété implicite d'`access_token`.

## Observer dans l'application

```bash
bin/console debug:firewall
bin/console debug:config security
bin/console debug:event-dispatcher --dispatcher=security.event_dispatcher.main
php bin/phpunit tests/Security
```

Une fois le firewall de l'atelier créé, remplacer `main` par son nom pour lire son dispatcher propre.
Lire `security-http/EventListener/UserCheckerListener.php`, `Authenticator/AccessTokenAuthenticator.php` et `AccessToken/HeaderAccessTokenExtractor.php` sous `vendor/symfony/`.
Les tests d'authentification envoient réellement les credentials ; `loginUser()` sauterait précisément les mécanismes étudiés.
Les compteurs d'effet métier doivent être observables après refus comme après succès.

## Documentation officielle

- [User checkers — Symfony 8.0](https://symfony.com/doc/8.0/security/user_checkers.html).
- [Authentification par access token — Symfony 8.0](https://symfony.com/doc/8.0/security/access_token.html).
- [Options SecurityBundle — Symfony 8.0](https://symfony.com/doc/8.0/reference/configuration/security.html).
- [Authenticators personnalisés — Symfony 8.0](https://symfony.com/doc/8.0/security/custom_authenticator.html).
