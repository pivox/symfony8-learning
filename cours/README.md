# Parcours Symfony 8 — maîtrise & certification

Un parcours personnel pour comprendre les mécanismes, les observer dans l’application, puis les mettre en pratique. Les supports sont en français ; les exercices sont à réaliser par l’apprenant. Les indices orientent la recherche sans fournir les solutions complètes.

## Versions et périmètre

État inspecté le **3 octobre 2026** : PHP **8.4.1**, Symfony **8.1.8**, PHPUnit **13.3.6**. Les versions exactes viennent de `composer.lock` et de `php bin/console about` ; `composer.json` demande PHP ≥ 8.4 et les composants Symfony 8.1.*.

La **certification Symfony 8 porte sur Symfony 8.0**, pas sur les ajouts 8.1. Chaque module distingue son socle des compléments récents. Les quiz sont des questions originales d’entraînement, pas des questions officielles. Une fonctionnalité observée dans `vendor/` 8.1 n’est pas automatiquement au programme 8.0. [Programme officiel](https://certification.symfony.com/exams/symfony.html).

Les ESI restent un complément pratique au module 05. Doctrine, Workflow et Lock/RateLimiter/Scheduler sont des extensions professionnelles proposées pour plus tard ; ils ne sont pas ajoutés à ce lot. Leur éventuel rattachement à la certification doit être vérifié dans le programme, composant par composant.

## Structure d’un module

- `README.md` : objectifs, prérequis, mécanismes, exemples courts, pièges, observation et références.
- `exercices.md` : progression guidée → autonome → diagnostic → défi, avec résultats attendus et indices.
- `quiz.md` : questions à faire sans documentation, puis réponses expliquées en bas du fichier.

Le code vit dans `src/<Module>/`, les tests dans `tests/<Module>/`. Les namespaces et chemins propres à chaque module sont précisés dans ses exercices. Les exemples des cours expliquent une API ; ils ne constituent pas les solutions des exercices.

## Modules et suivi personnel

**Lecture** et **quiz** sont à renseigner par l’apprenant. **Exercices** : ⬜ non commencés dans le dépôt, 🟨 partiels, ✅ tous les critères couverts. **Tests** : 🟨 tests écrits sur une partie seulement, résultat à vérifier à chaque modification ; ✅ critères du module entièrement testés et passants ; — pas encore de réalisation testée. Un test vert ne valide pas un exercice absent.

| # | Module / cours | Lecture | Exercices | Tests | Quiz |
|---|---|---|---|---|---|
| 01 | [PHP 8.4 & PSR](01-php84-psr/README.md) | ⬜ | 🟨 | 🟨 | — |
| 02 | [Kernel & architecture](02-kernel-architecture/README.md) | ⬜ | 🟨 | 🟨 | — |
| 03 | [DependencyInjection avancée](03-dependency-injection/README.md) | ⬜ | ⬜ | — | — |
| 04 | [HTTP, contrôleurs & sessions](04-http-rfc9110/README.md) | ⬜ | ⬜ | — | — |
| 05 | [Cache HTTP](05-cache-http/README.md) | ⬜ | ⬜ | — | — |
| 06 | [Security](06-security/README.md) | ⬜ | ⬜ | — | — |
| 07 | [Serializer avancé](07-serializer/README.md) | ⬜ | ⬜ | — | — |
| 08 | [Messenger](08-messenger/README.md) | ⬜ | ⬜ | — | — |
| 09 | [Twig & Forms](09-twig-forms/README.md) | ⬜ | ⬜ | — | — |
| 10 | [Composants & EventDispatcher](10-composants/README.md) | ⬜ | ⬜ | — | — |
| 11 | [Routing avancé](11-routing/README.md) | ⬜ | ⬜ | — | — |
| 12 | [Validator approfondi](12-validator/README.md) | ⬜ | ⬜ | — | — |
| 13 | [Console](13-console/README.md) | ⬜ | ⬜ | — | — |
| 14 | [Tests & diagnostic](14-tests-diagnostic/README.md) | ⬜ | ⬜ | — | — |
| 15 | [HttpClient](15-http-client/README.md) | ⬜ | ⬜ | — | — |
| 16 | [Cache applicatif](16-cache-applicatif/README.md) | ⬜ | ⬜ | — | — |
| 17 | [Configuration & environnements](17-configuration/README.md) | ⬜ | ⬜ | — | — |
| 18 | [Translation & localisation](18-translation/README.md) | ⬜ | ⬜ | — | — |
| 19 | [Mailer](19-mailer/README.md) | ⬜ | ⬜ | — | — |

Les cases de lecture sont non renseignées, pas une affirmation que le cours n’a pas été lu. Note le score du quiz sous la forme `bonnes réponses / total`, avec sa date, seulement après l’avoir passé.

Lors de la prise de connaissance initiale : **11 tests passants, 19 assertions**. Ce résultat historique est à revérifier après chaque modification du code. Les réalisations PHP 8.4 ne couvrent pas encore toutes les consignes ; pour le Kernel, les tests portent sur une route absente et `catch: false`. Les nouveaux supports ne marquent aucun exercice supplémentaire comme réalisé.

## Ordre conseillé

Les numéros restent stables pour conserver les repères existants. L’ordre pédagogique peut être différent :

1. **Fondations** : 01 → 02 → 03 ; lire en parallèle l’introduction de 14 pour écrire les premiers tests et les sections de configuration de 17.
2. **Requêtes et données** : 04 → 11 → 12 → 07. Faire les tests HTTP correspondants du module 14.
3. **Interface et accès** : parcours Twig de 09 → 06 → parcours Forms de 09, puis 18.
4. **Services et performance** : 10 → 15 → 05 → 16. Distinguer cache de valeurs et cache de réponses.
5. **Traitements** : 13 → 08 → 19 ; terminer les laboratoires de diagnostic de 14 et de préparation prod de 17.

Le premier lot à pratiquer après tes réalisations actuelles reste **Kernel, DI et HTTP**, complété par **Routing, Validator et Tests**. Les renvois servent à acquérir un prérequis précis ; il n’est pas nécessaire de terminer tous les défis des modules cités avant de commencer.

## Méthode de travail

1. Lire les objectifs, puis expliquer le mécanisme avec ses propres mots.
2. Observer les services, routes ou événements existants avec les commandes proposées.
3. Écrire un test qui échoue pour le comportement attendu ; coder ensuite l’exercice.
4. Vérifier succès, erreur et cas limite ; lire le journal et le profiler disponible en cas d’échec.
5. Faire le quiz sans consulter les réponses. Pour chaque erreur, noter la règle et une preuve dans la documentation ou le code installé.
6. Réaliser le défi, puis mettre à jour les quatre dimensions du suivi.

Pour demander de l’aide : « indice module 12 exercice 4 ». Premier niveau : direction de recherche ; deuxième : API et observation utile ; troisième : squelette ciblé sur demande. Une correction complète reste une demande explicite.

Les tests d’objets purs peuvent étendre `TestCase`, ceux des services `KernelTestCase`, ceux des actions HTTP `WebTestCase`. Les exercices de composant isolé l’indiquent. Aucun test ne doit dépendre d’une API publique, d’un email réel ou d’un temps d’attente inutile.

## Préparer l’environnement au bon moment

Déjà présents : Twig/Form, Security, Serializer/Validator, Messenger, HttpClient, Cache, Clock et les composants du module 10. Doctrine ORM/ODM et API Platform sont absents ; les scénarios utilisent des objets ou fixtures en mémoire.

Les tests fonctionnent déjà avec PHPUnit, BrowserKit et CssSelector. `symfony/test-pack` n’est pas déclaré ; pour préparer ou compléter l’outillage conformément aux conventions du projet, utiliser `composer require --dev symfony/test-pack` via Flex. Cela n’a pas été exécuté pour cet enrichissement documentaire.

| Besoin | Préparation lors de l’exercice |
|---|---|
| Profiler / barre de debug (module 14) | `composer require --dev symfony/profiler-pack` |
| Traduction (module 18) | `composer require symfony/translation` |
| Envoi d’emails simulé (module 19) | `composer require symfony/mailer`, puis transport nul dans le laboratoire |
| API PSR-16 pratique facultative (module 16) | `composer require psr/simple-cache` |

Le profiler n’est pas installé initialement : ne suppose pas que `/_profiler` existe. Après une installation, contrôler la recette et la configuration effective avant de commencer l’exercice. Les ajouts passent par Composer/Flex ; aucune modification manuelle de la liste des bundles n’est requise par les supports.

## Commandes de référence

Utiliser `symfony serve -d` et `symfony console ...` si la CLI Symfony est disponible ; sinon `php bin/console ...` pour les commandes applicatives.

```bash
php bin/console about
php bin/console debug:router
php bin/console debug:container
php bin/console debug:autowiring
php bin/console lint:container
php bin/console lint:yaml config/
php bin/console lint:twig templates/
php bin/phpunit
```

Pour un module déjà codé : `php bin/phpunit tests/Php84` ou `php bin/phpunit tests/Architecture`. Les autres dossiers de tests sont à créer en pratiquant les exercices.
