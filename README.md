# Apprendre Symfony 8

Un parcours en français pour comprendre Symfony, observer son fonctionnement et pratiquer avec des exercices progressifs. Le dépôt contient une application de laboratoire et les supports ; les solutions d’exercices sont à écrire par l’apprenant.

## Commencer

- [Parcours et suivi des 19 modules](cours/README.md) : cours, approfondissements, exercices et quiz corrigés.
- [Méthode et diagnostic](cours/methode-et-diagnostic.md) : organiser une séance, choisir un test et analyser un échec.
- [Quatre ateliers transversaux](cours/ateliers/README.md) : API de catalogue, formulaire de devis, import Console et traitement asynchrone.

La progression commence par PHP, le Kernel et l’injection de dépendances, puis aborde HTTP, les données, les interfaces et les traitements. Les indices orientent la recherche sans donner les solutions complètes. Les quiz sont des entraînements originaux, pas des questions officielles de certification.

## Environnement

Le projet demande **PHP ≥ 8.4** et **Symfony 8.1.*** ; les versions verrouillées et vérifiées le 11 octobre 2026 sont Symfony **8.1.8** et PHPUnit **13.3.6**. Le parcours distingue le socle Symfony 8.0 des compléments 8.1.

Depuis un clone disposant de PHP et Composer :

```bash
composer install
php bin/console about
php bin/phpunit
```

Pour lancer l’application avec la CLI Symfony :

```bash
symfony serve -d
symfony console debug:router
```

Sans la CLI Symfony, les commandes applicatives utilisent `php bin/console`. Les tests HTTP BrowserKit n’ont pas besoin d’un serveur lancé. Les prérequis facultatifs, comme Mailer ou Translation, sont indiqués dans leurs modules et s’installent avec Composer/Flex au moment de pratiquer.

## Organisation et vérification

Les supports vivent dans `cours/`. Le code d’exercice va dans `src/<Module>/` et les tests dans `tests/<Module>/` ; ces réalisations sont volontairement ignorées par Git. Un nouveau clone ne contient donc pas les réponses ni tous les tests personnels de l’apprenant.

```bash
symfony console lint:container
symfony console lint:yaml config/
symfony console lint:twig templates/
php bin/phpunit
```

Les conventions de contribution sont dans [AGENTS.md](AGENTS.md). Les critères de réussite figurent dans chaque exercice ; le nombre de tests verts ne remplace pas la vérification de ces critères.
