# Parcours Symfony — Implementation Plan

> **For agentic workers:** utiliser dispatching-parallel-agents pour les lots documentaires indépendants, puis une relecture commune.

**Goal:** Enrichir le parcours validé avec 19 modules exploitables sans résoudre les exercices.

**Architecture:** Documentation Markdown dans les dossiers pédagogiques existants. Aucun changement du code PHP, des tests ou des dépendances.

**Tech Stack:** Symfony 8.1.8, PHP 8.4.1, PHPUnit 13.3.6 ; socle certification Symfony 8.0.

## Tâches

- [x] Enrichir `cours/01-php84-psr/` à `cours/05-cache-http/` : objectifs, prérequis, compléments, cas limites, indices, quiz et défi ; préserver les exercices existants.
- [x] Enrichir `cours/06-security/` à `cours/10-composants/` selon les mêmes règles.
- [x] Créer les trois documents dans `cours/11-routing/`, `cours/12-validator/`, `cours/13-console/`, `cours/14-tests-diagnostic/`.
- [x] Créer les trois documents dans `cours/15-http-client/`, `cours/16-cache-applicatif/`, `cours/17-configuration/`, `cours/18-translation/`, `cours/19-mailer/`.
- [x] Réviser `cours/README.md` : liens, ordre pédagogique, distinction des versions et suivi multidimensionnel.
- [x] Relire les contenus et vérifier les API sensibles contre `vendor/` et les sources officielles ; corriger les incohérences.
- [x] Vérifier structure des 19 dossiers, liens Markdown locaux, blocs de code, numérotation des quiz et correspondance questions/réponses.
- [x] Exécuter `php bin/phpunit` (référence : 11 tests, 19 assertions) et `git diff --check` ; comparer les empreintes de src/tests/config et dépendances à `/tmp/tuto-before-enrichment.json`.

## Acceptation

Les 19 modules sont navigables depuis le sommaire ; les nouveaux cours proposent chacun huit exercices avec résultats attendus et quiz expliqués. Les anciens contenus sont conservés et approfondis. Aucun état de lecture, de quiz ou d’achèvement n’est attribué sans preuve ; aucun code de solution n’est ajouté.


## Résultat des vérifications

- 19 modules, 58 documents dans `cours/`, 339 questions avec corrections correspondantes et 162 liens locaux valides.
- Blocs Markdown équilibrés et absence d’espaces finaux ; 18 exemples PHP des nouveaux cours validés syntaxiquement (dont deux attributs replacés sur une déclaration de test).
- Relecture croisée et sondes PHP ciblées : HttpClient sans réseau, décodage invalide, TTL avec horloge partagée, recalcul forcé et valeur nulle en cache ; autres sondes réalisées par les relecteurs sur Validator, Console, Serializer, PropertyAccess, Clock, Process et EventDispatcher.
- Les 38 fichiers de code/configuration/dépendances présents au début conservent leurs empreintes. Aucun paquet installé et aucun exercice résolu.
- `git diff --check` passe. Les documents non suivis par Git ont également été contrôlés directement.
- `php bin/phpunit` a été exécuté : 11 tests, 12 assertions, 3 erreurs lors du démarrage du container. Un fichier nouveau et vide `src/Architecture/EventListener/ResponseHeaderListener.php` est apparu pendant le travail, hors modifications documentaires ; Symfony attend la classe correspondante. Le fichier a été laissé intact. Le résultat initial de 11 tests passants / 19 assertions n’est donc pas présenté comme le résultat final.
