# Parcours Symfony 8 — maîtrise & certification

Chaque module = un dossier `cours/NN-sujet/` contenant :
- `README.md` : notions clés, pièges certification, liens doc officielle
- `exercices.md` : exercices progressifs (à coder dans `src/` + tests dans `tests/`)
- `quiz.md` : questions type certification (réponses en bas)

Le code des exercices vit dans l'application : `src/<Module>/...`, testé dans `tests/<Module>/...`.
Lancer les tests : `php bin/phpunit` (ou `php bin/phpunit tests/<Module>`).

| #  | Module | Contenu | État |
|----|--------|---------|------|
| 01 | PHP 8.4 & PSR | property hooks, asymmetric visibility, `new` sans parenthèses, lazy objects, `array_find*` ; PSR-3/4/6/7/11/14/15/16/17/18/20 | ⬜ |
| 02 | Kernel / architecture | Runtime, `public/index.php`, HttpKernel::handle, événements `kernel.*` (request, controller, controller_arguments, view, response, finish_request, terminate, exception), ArgumentResolver, sous-requêtes, FrameworkBundle | ⬜ |
| 03 | DependencyInjection avancée | CompilerPass, tags, ServiceLocator, ServiceSubscriber, `#[AutowireLocator]`/`#[AutowireIterator]`, décoration (`#[AsDecorator]`, priorité, `.inner`), factories, lazy services, `#[Autoconfigure]`, `#[When]` | ⬜ |
| 04 | HTTP (RFC 9110) | méthodes sûres/idempotentes, codes de statut, content negotiation, cookies (SameSite, Secure…), `#[MapRequestPayload]`, `#[MapQueryString]`, validation | ⬜ |
| 05 | Cache HTTP | expiration (`max-age`, `s-maxage`), validation (ETag, Last-Modified, 304), `#[Cache]`, HttpCache reverse proxy, ESI, `Vary` | ⬜ |
| 06 | Security | firewalls, Authenticator, Passport, Badges, UserProvider, Voters, stratégies d'AccessDecision, `#[IsGranted]`, événements de sécurité | ⬜ |
| 07 | Serializer avancé | Normalizer/Encoder, contexte, groupes, `#[SerializedName]`, `#[MaxDepth]`, références circulaires, normalizers custom, `ObjectNormalizer` vs autres | ⬜ |
| 08 | Messenger | bus multiples (command/query/event), middleware, stamps, transports, routing, retry strategy, failure transport, `#[AsMessageHandler]` | ⬜ |
| 09 | Twig & Forms | Twig : héritage, `embed`, filtres/fonctions/extensions, escaping ; Forms : FormType, événements de form, DataTransformer, `inherit_data`, CollectionType, thèmes | ⬜ |
| 10 | Composants | Clock (`MockClock`), PropertyAccess, Finder, Filesystem, Mime, Process | ⬜ |

## Méthode
1. Lire le README du module (et la doc officielle liée).
2. Faire les exercices en TDD : écrire/compléter le test, puis le code.
3. Explorer avec les commandes de debug : `debug:container`, `debug:event-dispatcher`, `debug:router`, `debug:autowiring`, `debug:config`, `debug:messenger`, `debug:firewall`.
4. Répondre au quiz sans regarder la doc, puis corriger.
5. Cocher l'état dans ce tableau (⬜ → 🟨 → ✅).
