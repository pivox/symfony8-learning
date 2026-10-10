# Quiz — Module 03 : DependencyInjection avancée

Une seule bonne réponse sauf mention *(plusieurs réponses)*. Réponses en bas.

---

**Q1.** Quelle phase de `PassConfig` est utilisée par défaut par `$container->addCompilerPass($pass)` ?
- A. `TYPE_OPTIMIZE`
- B. `TYPE_BEFORE_OPTIMIZATION`
- C. `TYPE_REMOVE`
- D. `TYPE_AFTER_REMOVING`

**Q2.** Quel attribut(s) a (ont) été **supprimé(s)** en Symfony 8.0 ? *(plusieurs réponses)*
- A. `#[TaggedIterator]`
- B. `#[TaggedLocator]`
- C. `#[AutowireIterator]`
- D. `#[AutowireLocator]`

**Q3.** Une interface porte `#[AutoconfigureTag('app.handler')]`. Que se passe-t-il ?
- A. Rien : l'attribut n'est valide que sur les services
- B. Chaque classe qui l'implémente et est enregistrée avec `autoconfigure: true` reçoit le tag `app.handler`
- C. Le tag est ajouté à tous les services de l'application
- D. Une erreur de compilation est levée

**Q4.** Que reçoit un service avec `#[AutowireIterator('app.handler')] iterable $handlers` ?
- A. Un `array` de services déjà instanciés
- B. Un itérable paresseux : chaque service est instancié quand il est parcouru
- C. Un `ServiceLocator`
- D. Un seul service

**Q5.** Quelle est la valeur par défaut de `excludeSelf` dans `#[AutowireIterator]` / `#[AutowireLocator]` ?
- A. `false`
- B. `true`
- C. `null`
- D. Dépend de l'environnement

**Q6.** Quel est le rôle de `#[AsTaggedItem(index: 'pdf', priority: 20)]` ?
- A. Définit la clé et la priorité du service dans les collections taguées (iterator/locator)
- B. Déclare un alias
- C. Rend le service public
- D. Marque le service comme lazy

**Q7.** Deux décorateurs de `Mailer` : D1 avec `priority: 10`, D2 avec `priority: 0`. Lequel est appliqué en premier (le plus proche du service d'origine) ?
- A. D2
- B. D1
- C. L'ordre alphabétique des classes
- D. Impossible de décorer deux fois

**Q8.** Dans un décorateur, comment injecter le service décoré ? *(plusieurs réponses)*
- A. `#[AutowireDecorated]`
- B. `#[Autowire(service: 'inner')]`
- C. `#[Decorated]`
- D. `@.inner` en YAML

**Q9.** Après décoration de `App\Mailer` par `App\LoggingMailer`, que représente l'id `App\Mailer` ?
- A. Le service d'origine
- B. Le décorateur (le service d'origine est renommé `App\LoggingMailer.inner` par défaut)
- C. Un alias vers `null`
- D. Il n'existe plus

**Q10.** Quelle nouveauté 8.1 permet de décorer tous les services portant un tag ?
- A. `#[AsDecorator(tag: ...)]`
- B. `#[AsTagDecorator]` (ou la resource tag `container.tag_decorator`)
- C. `#[DecorateAll]`
- D. `decorates: '*'`

**Q11.** Que fait `#[Lazy]` sur une classe de service ?
- A. Le service est supprimé du container
- B. Le service est injecté sous forme de proxy/ghost, l'instance réelle étant créée au premier accès
- C. Le service est créé à chaque appel
- D. Le service est marqué `synthetic`

**Q12.** Que fait `#[When(env: 'dev')]` sur une classe ?
- A. Le service n'est enregistré que dans l'environnement `dev`
- B. Le service est enregistré partout sauf `dev`
- C. Le service est public en `dev`
- D. Rien : c'est un attribut de test

**Q13.** Quel est l'effet de `#[WhenNot(env: 'prod')]` ?
- A. Enregistré seulement en prod
- B. Enregistré dans tous les environnements sauf prod
- C. Désactive l'autowiring
- D. Équivalent à `#[Exclude]`

**Q14.** Quelles méthodes expose `ServiceLocator` (PSR-11) ? *(plusieurs réponses)*
- A. `get()`
- B. `has()`
- C. `set()`
- D. `getProvidedServices()` (via `ServiceProviderInterface`)

**Q15.** Un service privé n'est référencé par personne. Que lui arrive-t-il ?
- A. Il est dans le container compilé, mais inaccessible
- B. Il est supprimé du container compilé (`RemoveUnusedDefinitionsPass`)
- C. Il devient public automatiquement
- D. Une erreur est levée

**Q16.** Comment enregistrer une compiler pass directement depuis le Kernel de l'application ?
- A. En surchargeant `Kernel::build(ContainerBuilder $container)`
- B. En l'ajoutant dans `config/packages/framework.yaml`
- C. En appelant `Kernel::addPass()`
- D. Impossible : seulement dans un bundle

---


## Questions complémentaires — mise en situation

Une seule bonne réponse par question.

**Q17.** Deux services implémentent une interface sans alias de choix. Un consommateur demande cette interface. Que faire ?
- A. Compter sur le nom alphabétique
- B. Rendre les deux services publics
- C. Définir explicitement un alias ou sélectionner le service
- D. Ajouter un tag quelconque

**Q18.** D20 priorité 20 et D0 priorité 0 décorent S. Quel graphe reçoit le client ?
- A. D20(D0(S))
- B. D0(D20(S))
- C. S(D0(D20))
- D. Deux décorateurs en parallèle

**Q19.** Logging décore Sms, sans nom interne explicite. Quel identifiant interne est généré ?
- A. Sms.inner
- B. Logging.inner
- C. inner global partagé
- D. Sms.public

**Q20.** Un privé non utilisé est supprimé. Que garantit le container de test ?
- A. Sa recréation automatique
- B. Un alias public pour tout le monde
- C. L’accès aux privés conservés, sans ressusciter les supprimés
- D. La désactivation de toutes les compiler passes

**Q21.** Le RewindableGenerator produit pour un itérateur tagué est-il à usage unique ?
- A. Oui, exactement un parcours
- B. Non, il peut être parcouru à nouveau
- C. C’est toujours un tableau
- D. Il instancie forcément tous les services dès l’injection

## Réponses

**Q1 — B.** `addCompilerPass(CompilerPassInterface $pass, string $type = PassConfig::TYPE_BEFORE_OPTIMIZATION, int $priority = 0)`.

**Q2 — A, B.** Supprimés en 8.0, remplacés par `#[AutowireIterator]` et `#[AutowireLocator]` (CHANGELOG DI 8.0).

**Q3 — B.** `AutoconfigureTag` sur une interface/classe parente → autoconfiguration des implémentations.

**Q4 — B.** C'est un `iterable` paresseux (`RewindableGenerator`), pas un tableau.

**Q5 — B.** `excludeSelf = true` : un service tagué ne se reçoit pas lui-même dans l'itérateur.

**Q6 — A.** Remplace en 8.1 les `defaultIndexMethod`/`defaultPriorityMethod` (dépréciés).

**Q7 — B.** Priorité plus haute = appliqué en premier (le plus interne) ; D2 est donc le plus externe. Lecture de `DecoratorServicePass` (file de priorité, plus haute extraite en premier).

**Q8 — A, D.** `#[AutowireDecorated]` ou `@.inner` en YAML. (`#[Decorated]` n'existe pas ; `service: 'inner'` ne désigne aucun service.)

**Q9 — B.** L'id d'origine désigne désormais le décorateur ; l'ancien reçoit `<id du décorateur>.inner` (ou `decoration_inner_name`).

**Q10 — B.** `#[AsTagDecorator]` / `container.tag_decorator` (CHANGELOG DI 8.1).

**Q11 — B.** Proxy/ghost, instanciation au premier accès ; s'appuie sur les lazy objects natifs PHP 8.4 quand possible.

**Q12 — A.** `#[When]` est répétable pour cibler plusieurs envs.

**Q13 — B.** `#[WhenNot]` : tous les envs sauf celui indiqué.

**Q14 — A, B, D.** PSR-11 : `get`, `has`. Pas de `set`. `getProvidedServices()` liste id → type.

**Q15 — B.** Les services privés inutilisés sont supprimés à la compilation. D'où le `has()` faux dans un test si rien ne le référence.

**Q16 — A.** `Kernel::build()` (protégé) ; dans un bundle, `AbstractBundle::build()`.

**Q17 — C.** Le choix doit être non ambigu. A n’est pas une règle d’autowiring, B change la visibilité pas la résolution, D ne choisit pas une implémentation scalaire.

**Q18 — B.** La priorité haute est appliquée la première, donc à l’intérieur. A inverse cet ordre, C fait du service originel un décorateur, D ne correspond pas à la composition séquentielle.

**Q19 — B.** Le nom est construit depuis l’identifiant du décorateur. A confond décoré et décorateur, C créerait des collisions, D n’est pas la convention de décoration.

**Q20 — C.** Le container de test facilite l’accès mais ne remplace pas les définitions supprimées. A et B inventent une disponibilité, D empêcherait de tester le graphe réellement compilé.

**Q21 — B.** Le wrapper permet la réitération paresseuse. A le confond avec un générateur nu déjà fermé ; C confond iterable et array ; D supprime à tort la résolution à la demande.

