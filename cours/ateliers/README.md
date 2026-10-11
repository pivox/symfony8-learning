# Ateliers transversaux

[Parcours](../README.md) · [Méthode et diagnostic](../methode-et-diagnostic.md)

Ces quatre ateliers relient plusieurs modules autour d’un résultat utilisable. Ils sont indépendants : pas besoin de terminer le premier pour commencer le troisième. Les fixtures, routes et contrats sont propres à chaque atelier, sans corrigé complet.

## Choisir un atelier

| Atelier | Résultat à construire | Prérequis principaux | Durée indicative |
|---|---|---|---|
| [01 — API de catalogue](01-api-catalogue.md) | API publique JSON, recherche validée et réponse conditionnelle | 03, 04, 05, 07, 11, 12, 14, 15 | 3–4 h |
| [02 — Devis par formulaire](02-formulaire-devis.md) | Formulaire Twig, calcul, session et récapitulatif | 03, 04, 09, 12, 14 | 3–4 h |
| [03 — Import Console](03-import-console.md) | Import CSV contrôlé, simulation et rapport déterministe | 01, 03, 10, 12, 13, 14 | 3–5 h |
| [04 — Traitement asynchrone](04-traitement-asynchrone.md) | API d’acceptation, message sérialisé et effet observé | 03, 04, 07, 08, 12, 14 | 4–5 h |

Les durées comprennent les tests et le diagnostic. Commencer par le premier jalon si le sujet est nouveau ; les variantes viennent après les critères principaux.

## Préparation commune

Le projet utilise PHP 8.4 et Symfony 8.1 ; les ateliers emploient des API du socle 8.0. Utiliser les attributs pour les routes, contraintes, handlers et commandes, et l’autowiring pour les services. Les contrôleurs étendent `AbstractController` et délèguent les calculs à des services.

Le périmètre est explicite : aucune base de données ni API Platform ; fixtures en mémoire, session pour le seul atelier 02, fichiers temporaires pour le 03. Les interfaces des ateliers 01 et 04 sont des API publiques de laboratoire ; le 02 utilise Twig/Form et le 03 Console. Aucun authentificateur n’est à ajouter. Les extensions authentifiées se pratiquent séparément dans le module Security.

Les composants nécessaires sont déjà installés. Pour compléter les outils de test selon les conventions du dépôt, utiliser `composer require --dev symfony/test-pack` via Flex au moment de pratiquer ; le métapaquet n’est pas déclaré, mais PHPUnit, BrowserKit et CssSelector sont présents. L’extension Mailer du dernier atelier est facultative et indique son installation.

Créer le code dans `src/Ateliers/<Nom>/`, les tests dans `tests/Ateliers/<Nom>/`, les templates du formulaire dans `templates/ateliers/devis/`. Préfixer routes et services dédiés avec le nom de l’atelier pour éviter les collisions. Vérifier leur découverte avec `debug:router` et `debug:container`.

Le dépôt ignore volontairement les réalisations sous `src/` et `tests/`. Les supports restent suivis, les réponses restent locales. Ne pas forcer leur ajout à Git pour valider l’atelier. Utiliser un répertoire temporaire par test pour les artefacts ; ne pas stocker de fixtures dans `vendor/` ou des secrets dans les fichiers suivis.

## Validation commune

Pour chaque atelier, remettre à toi-même quatre éléments :

1. Les tests du contrat nominal, des refus et des effets absents.
2. Une courte carte « entrée → validation → service → sortie/effet ».
3. Une fiche d’incident reproductible à partir du défaut proposé.
4. Une limite technique explicitée : durabilité, livraison, cache partagé ou concurrence selon l’atelier.

Les tests HTTP étendent `WebTestCase`, les tests de services `KernelTestCase`. Préparer les doubles avant de résoudre les consommateurs ; les réinstaller après redémarrage du kernel si nécessaire. Les files en mémoire, les sessions et les caches demandent une isolation explicite entre scénarios.

```bash
php bin/phpunit tests/Ateliers
symfony console lint:container --env=test
symfony console lint:yaml config/
symfony console lint:twig templates/
```

Le dossier de tests n’existe qu’après réalisation. Aucun appel réseau public, envoi d’email réel ou attente murale n’est nécessaire.

## Suivi à remplir après pratique

| Atelier | Contrat nominal | Refus et limites | Diagnostic expliqué | Date / preuve |
|---|---|---|---|---|
| 01 | ⬜ | ⬜ | ⬜ | — |
| 02 | ⬜ | ⬜ | ⬜ | — |
| 03 | ⬜ | ⬜ | ⬜ | — |
| 04 | ⬜ | ⬜ | ⬜ | — |

Une coche nécessite une preuve de ta réalisation ; la présence de cet énoncé ne valide aucun atelier.
