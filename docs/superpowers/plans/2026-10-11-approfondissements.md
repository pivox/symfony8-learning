# Approfondissement du parcours — plan documentaire

> **For agentic workers:** lots documentaires indépendants avec la compétence `dispatching-parallel-agents`, puis relecture commune et vérification avant publication.

**Goal:** Enrichir les 19 modules et proposer des ateliers transversaux, puis pousser les supports sur la branche courante, conformément à la demande de l’utilisateur.

**Architecture:** Conserver les cours, exercices et quiz existants. Ajouter un `approfondissement.md` par module, deux exercices par module et quatre ateliers indépendants. Garder des énoncés français avec critères mesurables et indices progressifs, sans corrigés complets.

**Tech Stack:** Documentation Markdown ; APIs vérifiées dans Symfony 8.1.8 installé, références Symfony 8.0 pour le socle du parcours, PHP 8.4.1 et PHPUnit 13.3.6.

## Travaux

- [x] Lire les consignes, les versions, les recettes et la structure des supports ; constater une branche `main` propre et synchronisée avec `origin/main`.
- [x] Enregistrer les empreintes des 46 fichiers applicatifs, tests, configuration et dépendances présents ; exécuter les tests et lints de référence.
- [x] Ajouter les six approfondissements et douze exercices des modules 01–06, puis leurs liens de navigation.
- [x] Ajouter les six approfondissements et douze exercices des modules 07–12, puis leurs liens de navigation.
- [x] Ajouter les sept approfondissements et quatorze exercices des modules 13–19, puis leurs liens de navigation.
- [x] Créer `cours/ateliers/README.md` et quatre ateliers : API de catalogue, formulaire de devis, import Console, traitement asynchrone.
- [x] Ajouter `cours/methode-et-diagnostic.md` ; relier les nouveaux supports depuis `cours/README.md` sans attribuer d’avancement à l’apprenant.
- [x] Relire les API sensibles, critères, prérequis, liens, blocs Markdown et numéros ; comparer les empreintes des fichiers applicatifs.
- [x] Exécuter `php bin/phpunit`, les lints Symfony et `git diff --check` ; enregistrer le résultat réel.

Étape de publication après ces contrôles : créer un commit limité aux supports, pousser `main` sur `origin`, puis vérifier le commit distant. Le résultat du push est à confirmer dans le compte rendu de livraison.

## Critères de livraison

19 compléments navigables ; 38 nouveaux exercices ; quatre ateliers avec données, étapes, scénarios d’erreur et grille de validation. Aucune dépendance installée pour cette livraison documentaire, aucune réalisation d’exercice modifiée. Les fichiers ignorés de l’apprenant ne sont pas ajoutés au commit.

## Vérification de départ — 11 octobre 2026

- `php bin/phpunit` : 20 tests, 40 assertions, succès.
- `symfony console lint:container`, `lint:yaml config/` et `lint:twig templates/` : succès.

## Vérification finale — 11 octobre 2026

- 19 approfondissements, 38 exercices supplémentaires et quatre ateliers ; 83 documents dans `cours/`, plus un README à la racine.
- 444 liens locaux vérifiés, 67 références externes répondant HTTP 200, blocs Markdown et numérotation vérifiés.
- 39 extraits PHP vérifiés syntaxiquement ; cinq fragments replacés dans leur contexte de classe, de déclaration ou de paramètre pour le contrôle.
- Sondes comportementales sur ETag/304/HEAD, mapping de query, transport sérialisé, horloge et CSV ; relectures croisées sur les 19 modules et les ateliers.
- Corrections issues de la relecture : restauration des hosts de confiance, formulation de l'incident de pagination et prérequis Egulias pour composer une adresse MIME.
- `php bin/phpunit` : **20 tests, 40 assertions**, succès sur les réalisations locales existantes. Ce résultat ne signifie pas que les nouveaux exercices ont été réalisés.
- Conteneur validé ; 13 fichiers YAML et un template Twig valides ; `git diff --check` réussi.
- Empreintes des 46 fichiers applicatifs/configuration/dépendances inchangées. Aucun paquet installé ni exercice résolu.
