# Quiz — Module 06 : Security

Une ou plusieurs bonnes réponses : c'est indiqué. Réponses en bas.

**Q1 (une réponse).** Combien de firewalls s'exécutent pour une requête donnée ?
A. Tous ceux dont le pattern matche B. Le premier dont le pattern matche C. Le dernier D. Celui nommé `main`

**Q2 (une réponse).** Quelle est la stratégie par défaut de l'`AccessDecisionManager` ?
A. `unanimous` B. `consensus` C. `affirmative` D. `priority`

**Q3 (une réponse).** Tous les voters s'abstiennent, config par défaut : résultat ?
A. Accès accordé B. Accès refusé C. Exception de configuration D. Dépend du rôle

**Q4 (plusieurs réponses).** Valeurs possibles de retour de `VoterInterface::vote()` :
A. `ACCESS_GRANTED` (1) B. `ACCESS_ABSTAIN` (0) C. `ACCESS_DENIED` (-1) D. `ACCESS_UNKNOWN` (2)

**Q5 (une réponse).** Stratégie `consensus`, 2 grants, 2 denies, option par défaut : résultat ?
A. Refusé B. Accordé (`allow_if_equal_granted_denied` = true) C. Exception D. Abstention

**Q6 (une réponse).** Stratégie `priority` : qui décide ?
A. Le voter ayant voté GRANTED en dernier B. Le premier voter qui ne s'abstient pas C. Le plus de votes D. Le voter `RoleVoter`

**Q7 (une réponse).** Dans une classe étendant `Voter`, si `supports()` renvoie `false` :
A. `ACCESS_DENIED` B. `ACCESS_ABSTAIN` C. Exception D. `voteOnAttribute()` est appelé quand même

**Q8 (une réponse).** `access_control` : deux règles matchent l'URL.
A. Les deux sont appliquées B. La première rencontrée est appliquée C. La plus spécifique est appliquée D. La dernière est appliquée

**Q9 (plusieurs réponses).** Éléments d'un `Passport` standard :
A. `UserBadge` B. `CredentialsInterface` (ex. `PasswordCredentials`) C. Badges optionnels (`CsrfTokenBadge`, `RememberMeBadge`...) D. Un `UserProviderInterface`

**Q10 (une réponse).** `onAuthenticationSuccess()` d'un authenticator retourne `null` : conséquence ?
A. Erreur B. La requête continue vers le contrôleur C. Redirection vers `/` D. Déconnexion

**Q11 (une réponse).** Un badge reste non résolu à la fin de `CheckPassportEvent` :
A. Ignoré B. Exception levée (échec d'authentification technique) C. Résolu automatiquement D. Avertissement uniquement

**Q12 (une réponse).** Quelle méthode a été **supprimée** de `UserInterface` en Symfony 8.0 ?
A. `getRoles()` B. `getUserIdentifier()` C. `eraseCredentials()` D. `getPassword()`

**Q13 (plusieurs réponses).** Événements du namespace `Symfony\Component\Security\Http\Event` :
A. `LoginSuccessEvent` B. `LoginFailureEvent` C. `LogoutEvent` D. `AuthenticationSuccessEvent`

**Q14 (une réponse).** Où s'exécute `#[IsGranted]` ?
A. Avant le routage B. Avant l'exécution du contrôleur (écouteur de `kernel.controller_arguments`) C. Après le contrôleur D. Dans le firewall

**Q15 (plusieurs réponses).** Quelles affirmations sur `lazy: true` sont vraies ?
A. L'authentification n'est vérifiée que si on accède à l'utilisateur/`isGranted()` B. Utile pour ne pas démarrer la session inutilement C. Désactive le firewall D. Équivalent à `stateless: true`

---

## Questions de mise en situation — 16 à 20

**Q16.** (1 réponse) Un authenticator retourne `supports() === false` pour une route protégée. Quelle conséquence est garantie ?

- A. La route devient publique
- B. Cet authenticator ignore la requête, les contrôles d’accès restent applicables
- C. Le contrôleur doit renvoyer 403
- D. Une session authentifiée est créée

**Q17.** (1 réponse) Un utilisateur pleinement authentifié, sans remember-me seul, manque de `ROLE_ADMIN`. Sans personnalisation, le refus donne :

- A. Une redirection obligatoire vers le login
- B. 401 dans tous les cas
- C. 403
- D. Une nouvelle tentative de mot de passe automatique

**Q18.** (1 réponse) Que prouve un test utilisant `loginUser()` ?

- A. Le mot de passe a été vérifié
- B. Le token CSRF du login est correct
- C. Le logout invalide fonctionne
- D. Le comportement de la route pour une identité injectée

**Q19.** (1 réponse) Après un logout protégé accepté, quelle assertion est la plus probante ?

- A. La route protégée refuse le même client à la requête suivante
- B. Le logout a seulement renvoyé une redirection
- C. Le bouton logout existe dans le HTML
- D. Le mot de passe de l’utilisateur a changé

**Q20.** (1 réponse) Pour un identifiant CSRF figurant dans `stateless_token_ids`, quelle affirmation est exacte ?

- A. Aucune protection CSRF ne subsiste
- B. L’égalité avec un secret de session est toujours le seul contrôle
- C. La validation peut utiliser les informations d’origine ou la double soumission
- D. Un identifiant utilisateur valide remplace les contrôles CSRF

## Réponses

1. **B**. Seul le premier firewall dont le pattern matche.
2. **C**. `new AffirmativeStrategy()` si aucune stratégie fournie (`AccessDecisionManager`).
3. **B**. `allow_if_all_abstain` vaut `false`.
4. **A, B, C**. Constantes de `VoterInterface`.
5. **B**. `allowIfEqualGrantedDeniedDecisions` vaut `true` par défaut dans `ConsensusStrategy`.
6. **B**. Premier non-abstention : GRANTED → accès, DENIED → refus.
7. **B**. `Voter::vote()` s'abstient quand `supports()` est faux.
8. **B**. Premier motif qui matche.
9. **A, B, C**. Le provider n'est pas dans le Passport (utilisé par `UserProviderListener` si le `UserBadge` n'a pas de loader).
10. **B**. `null` = la requête continue.
11. **B**. Garde-fou : un badge non résolu lève une exception.
12. **C**. Supprimée en 8.0.
13. **A, B, C**. `AuthenticationSuccessEvent` est dans `Security\Core\Event`.
14. **B**. `IsGrantedAttributeListener` (avant le contrôleur).
15. **A, B**. `lazy` n'est ni `stateless` ni une désactivation.


## Réponses raisonnées — 16 à 20

16. **B.** Le mécanisme se déclare non concerné. A confond sélection et autorisation ; C dépend de l’identité et de l’entry point ; D exige une authentification réelle.
17. **C.** L’identité est suffisante, les droits ne le sont pas. A et B correspondent à d’autres chemins de démarrage d’authentification ; D n’est pas le rôle de l’autorisation.
18. **D.** L’utilitaire contourne l’authenticator. A et B demandent une vraie requête de login ; C exige son propre scénario de sortie.
19. **A.** La requête suivante démontre l’effet sur l’authentification. B et C ne le prouvent pas ; D ne fait pas partie d’une déconnexion.
20. **C.** La stratégie sans état protège toujours la requête. A est faux ; B décrit uniquement un autre mécanisme ; D confond authentification et intention de soumission.
