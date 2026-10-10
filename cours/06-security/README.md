# Module 06 — Security

## Objectifs, prérequis et repères

> Application : **Symfony 8.1.8, PHP ≥ 8.4**. La [certification Symfony](https://certification.symfony.com/exams/symfony.html) cible **8.0** : les mentions **8.1 uniquement** sont des compléments pour cette application, pas des acquis à attribuer à 8.0.

À la fin du module, tu dois pouvoir tracer une authentification jusqu'au token, distinguer absence d'identité, identifiants invalides et droits insuffisants, puis prouver la protection CSRF et la déconnexion par HTTP.

Prérequis : cycle HttpKernel, services et événements (modules 02–04), notions de session et cookies ; [routage](../11-routing/README.md) pour les URLs protégées. SecurityBundle et CSRF sont installés. Le firewall `main` actuel est lazy et sans authenticator : les scénarios ci-dessous décrivent la configuration à construire dans les exercices. Utiliser des utilisateurs en mémoire : **Doctrine n'est pas installé**. `loginUser()` est adapté aux tests d'autorisation, mais ne teste ni les credentials ni l'authenticator.

Applique la [méthode de travail commune](../README.md) : prédire, exécuter, observer, expliquer. Les exemples ci-dessous illustrent un mécanisme ; les exercices restent à réaliser. Le profiler n'est pas installé : utiliser les commandes et les tests proposés. Son installation facultative passerait par `composer require --dev symfony/profiler-pack`.

Packages déjà installés : `symfony/security-bundle` (qui tire `security-core`, `security-http`, `security-csrf`, `password-hasher`). Non installé : `symfony/rate-limiter` (requis par `login_throttling`) → `composer require symfony/rate-limiter` si tu veux l'exercice sur la limitation de tentatives.

Deux questions distinctes :
- **Authentification** : *qui es-tu ?* (firewall + authenticators → un `TokenInterface` contenant un `UserInterface`).
- **Autorisation** : *as-tu le droit ?* (`access_control`, `#[IsGranted]`, `isGranted()`, voters, AccessDecisionManager).

## 1. Firewalls

Un firewall = une portion de l'app (sélectionnée par `pattern`/`request_matcher`/`host`...), avec ses authenticators, son user provider, son contexte de session. **Seul le premier firewall dont le pattern matche s'exécute.**

```yaml
# config/packages/security.yaml
security:
    providers:
        app_users: { memory: null }         # ajouter des users ou un provider custom ; aucun Doctrine ici
    firewalls:
        dev:
            pattern: ^/(_profiler|_wdt|assets)/
            security: false                  # aucun mécanisme de sécurité
        main:
            lazy: true                       # ne charge l'utilisateur que si on le demande
            provider: app_users
            stateless: false                 # true => pas de session (API)
            form_login: { login_path: app_login, check_path: app_login }
            logout: { path: app_logout }
            remember_me: { secret: '%kernel.secret%' }
            # switch_user: true
```

- `lazy: true` : le firewall ne démarre pas la session/authentification tant qu'on n'accède pas à l'utilisateur ou à `isGranted()` (conserve la possibilité de cache HTTP : voir module 05).
- `stateless: true` : la sécurité ne conserve pas l’authentification en session ; chaque requête protégée doit fournir ses credentials (token/API). Cela n’interdit pas à un autre service de manipuler une session.
- `entry_point` : qui renvoie la réponse « 401/redirect login » à un anonyme refusé (obligatoire quand plusieurs authenticators interactifs).
- `user_checker` : service implémentant `UserCheckerInterface` (`checkPreAuth`, `checkPostAuth(UserInterface $user, ?TokenInterface $token = null)`).
- Types d'authenticators natifs enregistrés par SecurityBundle (vérifié) : `form_login`, `json_login`, `http_basic`, `remember_me`, `x509`, `remote_user`, `login_link`, `access_token` (token handlers : service, OIDC, OIDC userinfo, CAS, OAuth2), `login_throttling`, `custom_authenticators`.

## 2. Authenticator, Passport, Badges

Depuis 5.x/6.0 il n'existe plus de « guard » ; tout est basé sur `AuthenticatorInterface` (vérifié dans `security-http`) :

```php
interface AuthenticatorInterface
{
    public function supports(Request $request): ?bool;          // null => « je ne sais pas » (lazy), false => ignore
    public function authenticate(Request $request): Passport;   // lève AuthenticationException en cas d'échec
    public function createToken(Passport $passport, string $firewallName): TokenInterface;
    public function onAuthenticationSuccess(Request $request, TokenInterface $token, string $firewallName): ?Response;
    public function onAuthenticationFailure(Request $request, AuthenticationException $exception): ?Response;
}
```

- Étendre `AbstractAuthenticator` (fournit `createToken`) ou `AbstractLoginFormAuthenticator`.
- `onAuthenticationSuccess/Failure` retournant `null` => la requête **continue** vers le contrôleur.

```php
public function authenticate(Request $request): Passport
{
    return new Passport(
        new UserBadge($request->request->get('email')),                 // identifiant
        new PasswordCredentials($request->request->get('password')),    // vérifié par CheckCredentialsListener
        [new CsrfTokenBadge('login', $request->request->get('_csrf_token')),
         new RememberMeBadge()],
    );
}
// authentification sans credentials (token API déjà validé) :
// new SelfValidatingPassport(new UserBadge($identifier, $userLoaderCallable));
```

- `Passport` = `UserBadge` + `CredentialsInterface` (`PasswordCredentials`, `CustomCredentials`) + badges optionnels.
- Badges natifs (vérifié) : `UserBadge`, `CsrfTokenBadge`, `RememberMeBadge`, `PasswordUpgradeBadge`, `PreAuthenticatedUserBadge`.
- Un badge est « résolu » par un écouteur de `CheckPassportEvent`. **Un badge non résolu à la fin de l'événement provoque une exception** : c'est ainsi que la sécurité est garantie (ex. `CsrfTokenBadge` ↔ `CsrfProtectionListener`, `UserBadge` ↔ `UserProviderListener`, `PasswordCredentials` ↔ `CheckCredentialsListener`).
- `UserBadge` : l'identifiant + (optionnel) un callable pour charger l'utilisateur ; sinon le **user provider** du firewall le charge (`loadUserByIdentifier`).
- `PasswordUpgradeBadge` : rehash automatique à `LoginSuccessEvent` (`PasswordMigratingListener`) si l'algorithme `auto` a évolué ; il faut que le provider implémente `PasswordUpgraderInterface`.

## 3. Événements de sécurité

Dans `Symfony\Component\Security\Http\Event` (vérifié) : `CheckPassportEvent`, `AuthenticationTokenCreatedEvent`, `LoginSuccessEvent`, `LoginFailureEvent`, `LogoutEvent`, `SwitchUserEvent`, `InteractiveLoginEvent`, `TokenDeauthenticatedEvent`, `LazyResponseEvent`. Et dans `security-core` : `AuthenticationSuccessEvent`, `VoteEvent`.

Ordre de l'authentification d'une requête : `supports()` → `authenticate()` (Passport) → `CheckPassportEvent` (priorités : throttling 2080, UserCheckerListener pré-auth 256, CSRF 512, etc.) → `createToken()` → `AuthenticationTokenCreatedEvent` → `AuthenticationSuccessEvent` (core, user checker post-auth) → `onAuthenticationSuccess()` → `LoginSuccessEvent` ; en cas d'exception : `LoginFailureEvent`. `LogoutEvent` : écouteurs pour CSRF, session, cookies, redirection (`DefaultLogoutListener`).

```php
#[AsEventListener]
public function onLoginSuccess(LoginSuccessEvent $event): void
{
    $event->getUser(); $event->getFirewallName(); $event->getPassport();
    $event->setResponse(new RedirectResponse('/'));   // remplacer la réponse
}
```

## 4. Users et providers

```php
class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    public function getRoles(): array { return ['ROLE_USER', ...]; }
    public function getUserIdentifier(): string { ... }
}
```

- `UserInterface` : `getRoles()`, `getUserIdentifier()`. **`eraseCredentials()` a été supprimé en 8.0** (vérifié dans `security-core/CHANGELOG.md`) : effacer les secrets via `__serialize()` si nécessaire.
- `UserProviderInterface` : `loadUserByIdentifier(string)`, `refreshUser(UserInterface)`, `supportsClass(string)`. Le refresh se fait à chaque requête (firewall à session) ; si l'utilisateur a changé (comparaison du token), il est **déconnecté**.
- `providers` : `memory`, `entity` (Doctrine, non installé ici), `ldap`, `chain`, service custom (`id`).
- Hash des mots de passe : `password_hashers` + `UserPasswordHasherInterface`, `bin/console security:hash-password`.

## 5. Autorisation : AccessDecisionManager, Voters, stratégies

`AuthorizationChecker::isGranted($attribute, $subject)` → `AccessDecisionManager::decide()` → interroge les voters **jusqu’à ce que la stratégie puisse décider** (`vote()` retourne `ACCESS_GRANTED = 1`, `ACCESS_ABSTAIN = 0`, `ACCESS_DENIED = -1`) → la **stratégie** combine les résultats.

Stratégies (vérifiées dans `security-core/Authorization/Strategy/`) :

| Stratégie (`security.access_decision_manager.strategy`) | Règle |
|---|---|
| `affirmative` (**défaut**, `new AffirmativeStrategy()` si aucune fournie) | accès accordé dès qu'**un** voter accorde ; refus si aucun grant et au moins un deny |
| `consensus` | majorité des grants vs denies (abstentions ignorées) ; égalité → `allow_if_equal_granted_denied` (défaut **true**) |
| `unanimous` | accordé s'il n'y a aucun deny et au moins un grant |
| `priority` | le **premier** voter qui ne s'abstient pas décide (priorité des voters = tag/`#[AsTaggedItem]`) |

- Si tous s'abstiennent : `allow_if_all_abstain` (défaut **false** → refus). Options : `strategy`, `service`, `strategy_service`, `allow_if_all_abstain`, `allow_if_equal_granted_denied`.

Voter custom :

```php
final class PostVoter extends Voter
{
    protected function supports(string $attribute, mixed $subject): bool
    {
        return \in_array($attribute, ['POST_EDIT'], true) && $subject instanceof Post;
    }

    protected function voteOnAttribute(string $attribute, mixed $subject, TokenInterface $token, ?Vote $vote = null): bool
    {
        $user = $token->getUser();
        return $user instanceof User && $subject->getAuthor() === $user;
    }
}
```

- `Voter::vote()` : `supports()` faux → **abstention** ; sinon `voteOnAttribute()` → `true` = GRANTED, `false` = DENIED.
- Voters natifs : `RoleVoter` (`ROLE_*`), `RoleHierarchyVoter` (hiérarchie `role_hierarchy`), `AuthenticatedVoter` (`IS_AUTHENTICATED_FULLY`, `IS_AUTHENTICATED_REMEMBERED`, `IS_AUTHENTICATED`), `ExpressionVoter`.
- Les voters custom sont autoconfigurés (tag `security.voter`). Priorité via `#[AsTaggedItem(priority: ...)]` (nouveau en **8.1** côté bundle).
- `Vote` et raisons de refus (7.3 ; signature exigée en 8.0) : paramètre optionnel `?Vote $vote` de `vote()`/`voteOnAttribute()` pour `addReason()` ; `AccessDecision` sur `isGranted()` et `Security::getAccessDecision()` (7.4).

## 6. Contrôle d'accès : `access_control`, `#[IsGranted]`, `isGranted()`

```yaml
access_control:        # le PREMIER motif qui matche gagne
    - { path: ^/admin, roles: ROLE_ADMIN }
    - { path: ^/api, roles: IS_AUTHENTICATED_FULLY, methods: [POST, PUT] }
    - { route: app_secret, allow_if: "is_granted('ROLE_ADMIN') or request.getClientIp() == '127.0.0.1'" }
```
Clés supportées : `path`, `route`, `host`, `port`, `ips`, `methods`, `request_matcher`, `requires_channel`, `allow_if`, `roles`/`attributes`.

```php
#[IsGranted('ROLE_ADMIN')]                                    // classe ou méthode
#[IsGranted('POST_EDIT', subject: 'post', statusCode: 404)]   // subject = nom d'argument (ou Expression/closure)
public function edit(Post $post): Response {}

$this->denyAccessUnlessGranted('POST_EDIT', $post);           // AbstractController
if ($security->isGranted('POST_EDIT', $post)) {}              // Symfony\Bundle\SecurityBundle\Security
```

`#[IsGranted]` (vérifié) : `attribute`, `subject`, `message`, `statusCode`, `exceptionCode`, `methods`. L'attribut s'applique **avant** l'exécution du contrôleur (`IsGrantedAttributeListener` sur `kernel.controller_arguments` ; en 8.1 il écoute aussi l'événement d'attribut `kernel.controller_arguments.<FQCN de IsGranted>`, cf. `ControllerAttributeEvent` dans le CHANGELOG http-kernel). En 8.1 : `this` est disponible dans l'expression du subject.

Autres attributs : `#[CurrentUser]` (injecte l'utilisateur), `#[IsCsrfTokenValid]`.

Refus : le contrôle d’accès produit une `AccessDeniedException`. Le listener de sécurité peut démarrer l’authentification pour une identité absente ou insuffisamment authentifiée (entry point : redirection / 401). Pour un utilisateur pleinement authentifié sans les droits requis : 403 par défaut. Un `statusCode` personnalisé sur `IsGranted` change cette réponse.

## 7. Role hierarchy
```yaml
security:
    role_hierarchy:
        ROLE_ADMIN: [ROLE_USER]
        ROLE_SUPER_ADMIN: [ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
```
`$user->getRoles()` ne contient PAS les rôles hérités : c'est `RoleHierarchyVoter` qui les résout. 8.1 : `RoleHierarchyInterface::getParentRoleNames()` ajouté.

## Ce qui change vs Symfony 6/7
- **8.0** : `UserInterface::eraseCredentials()` et `TokenInterface::eraseCredentials()` supprimés ; l'option `security.erase_credentials` est dépréciée en 8.1.
- 8.0 : l'option `hide_user_not_found` supprimée → `expose_security_errors`.
- 8.0 : `AccessDecisionStrategyInterface::decide()`, `AccessDecisionManagerInterface::decide()`, `isGranted()`, `VoterInterface::vote()` ont un argument additionnel (`AccessDecision` / `Vote`) ; `UserCheckerInterface::checkPostAuth()` reçoit le `$token`.
- 8.0 : l'utilisateur FQCN n'est plus dans le cookie remember-me ; `RememberMeToken::getSecret()` supprimé.
- 8.0 : autowiring alias `RateLimiterFactory` supprimés → `RateLimiterFactoryInterface`.
- 8.1 : `#[AsTaggedItem]` pour la priorité des voters ; `this` dans les expressions `#[IsGranted]` ; enums dans `SignatureHasher`.
- Déjà depuis 6.0 : plus de « Guard », `AuthenticatorInterface` + Passport ; `getUserIdentifier()` (plus `getUsername`) ; `isGranted()` sur `Security` du SecurityBundle (`Symfony\Bundle\SecurityBundle\Security`, plus `Symfony\Component\Security\Core\Security`).

## Piège certification
- Stratégie par défaut = **affirmative**, et si tous les voters s'abstiennent le refus est la règle (`allow_if_all_abstain: false`).
- `access_control` : **premier match gagne** ; l'ordre compte (règle spécifique avant règle générique).
- `access_control` s'exécute au niveau firewall (`AccessListener`) ; `#[IsGranted]` au niveau contrôleur ; un firewall `security: false` n'applique **ni** authentification **ni** `access_control`.
- Un voter qui ne supporte pas l'attribut s'**abstient** (≠ refuse).
- `authenticate()` doit lever une `AuthenticationException` (ou `CustomUserMessageAuthenticationException` pour un message à afficher) ; `onAuthenticationSuccess` renvoyant `null` poursuit la requête.
- `supports()` qui renvoie `false` : l'authenticator est ignoré ; sinon `authenticate()` est appelé.
- Badge non résolu => exception : on ne peut pas « oublier » une vérification.
- `lazy` firewall : l'utilisateur n'est chargé qu'à la demande.
- Les rôles doivent commencer par `ROLE_` pour être traités par `RoleVoter`.

## Observer dans l'app
- `bin/console debug:firewall` puis `debug:firewall main` : authenticators, listeners, contexte.
- `bin/console debug:config security`, `debug:config security access_control`.
- `bin/console debug:event-dispatcher Symfony\\Component\\Security\\Http\\Event\\CheckPassportEvent` (ou `LoginSuccessEvent`, `LogoutEvent`) pour voir ordre et priorités des écouteurs.
- `bin/console debug:container --tag=security.voter` : liste des voters ; `debug:container security.access.decision_manager`.
- `bin/console debug:security:role-hierarchy` (7.4+, sortie Mermaid).
- `bin/console security:hash-password`.
- Après installation facultative du profiler : profiler → onglet **Security** : utilisateur, rôles, firewall, décisions de chaque voter (c'est `TraceableVoter` / `TraceableAccessDecisionManager`), stratégie utilisée.
- Tests : `$client->loginUser($user)` (`KernelBrowser`, test env) ; `$this->assertResponseStatusCodeSame(403)`; pour un voter, test unitaire direct avec un `TokenInterface` mocké (ou `UsernamePasswordToken`).

## Doc officielle
- https://symfony.com/doc/current/security.html
- https://symfony.com/doc/current/security/custom_authenticator.html
- https://symfony.com/doc/current/security/voters.html
- https://symfony.com/doc/current/security/access_control.html
- https://symfony.com/doc/current/security/expressions.html
- https://symfony.com/doc/current/security/user_providers.html
- https://symfony.com/doc/current/security/passwords.html
- https://symfony.com/doc/current/reference/configuration/security.html
- https://symfony.com/doc/current/components/security/authorization.html (AccessDecisionManager et stratégies)

## 8. Lire une authentification comme un scénario HTTP

Le firewall ne prouve pas à lui seul une identité : il choisit les mécanismes qui peuvent la prouver. `supports() === false` signifie « cet authenticator ne traite pas cette requête », **pas** « autoriser l'accès ». Une route protégée déclenchera ensuite l'entry point si aucune identité acceptable n'a été établie. À l'inverse, une tentative invalide passe par `onAuthenticationFailure()` ; n'attends donc pas nécessairement la même réponse que pour une requête sans tentative.

| Requête dans un scénario API protégé | Résultat choisi pour le contrat | Preuve à relever |
|---|---|---|
| Aucun token | 401 via un entry point JSON configuré | aucun handler métier appelé |
| Token inconnu | 401 via l'échec d'authentification | `LoginFailureEvent`, message public générique |
| Token valide, rôle insuffisant | 403 | identité reconnue, voter ou règle en refus |
| Token valide, droit accordé | 200 | action appelée une fois |

Exemple minimal de contrôle, une fois une route et un authenticator configurés :

```php
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[IsGranted('ROLE_EDITOR')]
public function publish(): Response
{
    return new Response('publication autorisée');
}
```

Le test d'autorisation peut utiliser `loginUser()`. Pour vérifier le cycle complet, envoyer réellement les credentials et conserver les cookies du même client sur la requête suivante. Un succès sur la requête de login ne prouve pas que la session est utilisable. Un firewall `stateless` n'enregistre pas l'authentification en session : le token doit revenir dans **chaque** requête.

Le Passport est une promesse à vérifier, pas un token final. Les listeners résolvent le `UserBadge`, vérifient les credentials et la protection CSRF, puis l'authenticator crée le token. `onAuthenticationSuccess()` construit une réponse éventuelle, ensuite `LoginSuccessEvent` permet encore de l'ajuster. Le callback peut retourner `null` pour continuer vers l'action.

### CSRF et logout : ne pas confondre identité et intention

Un cookie de session est envoyé automatiquement par le navigateur : une identité reconnue ne démontre pas que l'utilisateur voulait exécuter l'action. Pour un formulaire de connexion natif, `form_login.enable_csrf: true` active la vérification ; pour un authenticator custom, le `CsrfTokenBadge` doit utiliser le **même identifiant de token** que le formulaire. `logout.enable_csrf: true` protège également la route de sortie. Utilise les fonctions Twig `csrf_token()` et `logout_path()` lorsque adaptées à la configuration ; ne fabrique pas un secret dans le contrôleur.

**Particularité de la recette locale** : `submit`, `authenticate` et `logout` sont déclarés dans `framework.csrf_protection.stateless_token_ids`. Leur validation peut s'appuyer sur l'origine HTTP ou une stratégie double soumission ; ce n'est pas forcément une comparaison avec un token stocké en session. Pour apprendre le mécanisme de session sans ambiguïté, utiliser un identifiant d'exercice distinct, par exemple `lesson_login`, absent de cette liste. Tester séparément ensuite les requêtes cross-origin avec la configuration sans état. Ne pas conclure qu'un token arbitraire prouve un bug si les vérifications d'origine autorisent la requête.

Le logout est intercepté par Security : un contrôleur métier n'a normalement aucun nettoyage à effectuer. Après une sortie valide, vérifier avec le **même client** que la route protégée redéclenche l'entry point. Une réponse 302 à `/logout` seule ne prouve rien ; la session, les cookies remember-me et la requête suivante font partie du scénario.

Observer : `bin/console debug:firewall main`, `debug:config security`, puis `debug:event-dispatcher --dispatcher=security.event_dispatcher.main`. Lire `security-http/Authentication/AuthenticatorManager.php` et `Firewall/ExceptionListener.php` dans `vendor/symfony/`. Comparer au [cycle de sécurité documenté en 8.0](https://symfony.com/doc/8.0/security.html).
