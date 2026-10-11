# Module 17 — Configuration & environnements

[Parcours](../README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Expliquer d'où vient une valeur, isoler dev/test/prod et préparer une application à démarrer avec une configuration reproductible. Prérequis : [Kernel](../02-kernel-architecture/README.md), [DI](../03-dependency-injection/README.md), [Console](../13-console/README.md).

**Versions :** dépôt Symfony 8.1.8 ; mécanismes de base étudiés pour Symfony 8.0. Dotenv et Flex sont installés. Aucune base de données, authentification ou plateforme de déploiement n'est nécessaire.

## 1. Quatre sources différentes

- `composer.json` exprime les contraintes et les scripts ; `composer.lock` fixe les versions résolues.
- `symfony.lock` indique les recettes Flex appliquées. Ce n'est pas un second verrou de versions des bibliothèques.
- `config/` configure les services et les extensions des bundles.
- Les variables d'environnement apportent les valeurs dépendantes de l'environnement ; elles ne remplacent pas la structure de configuration.

Pour ajouter une capacité, utilise `composer require` et laisse Flex appliquer sa recette. Une commande Maker visible ne prouve pas que toutes ses dépendances sont installées : vérifie les paquets et l'aide de la commande.

## 2. Comprendre Dotenv

Avec le chargement standard et sans surcharge explicite des variables système, l'ordre des fichiers va du moins prioritaire au plus prioritaire :

```text
.env → .env.local → .env.<environnement> → .env.<environnement>.local
```

Les véritables variables du processus conservent normalement la priorité. En environnement `test`, `.env.local` est ignoré ; `.env.test.local` peut toutefois s'appliquer. Les tests doivent donc maîtriser les variables qu'ils comparent.

Un éventuel `.env.local.php`, produit par un dump, change le chemin de chargement de `bootEnv()` : vérifie s'il existe avant d'expliquer pourquoi une modification d'un fichier texte n'est pas visible. Ne publie pas ce fichier s'il contient des secrets.

`APP_ENV` sélectionne l'environnement ; `APP_DEBUG` contrôle le debug. Ils sont liés par les valeurs par défaut du démarrage, mais restent deux réglages distincts. Le Kernel du dépôt n'accepte que `dev`, `test` et `prod`.

## 3. Paramètres typés et injection

Une variable d'environnement arrive généralement comme chaîne. Les processeurs tels que `int:`, `bool:` ou `json:` servent à demander une interprétation explicite.

Exemple de dépendance scalaire déclarée dans la classe :

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

final class BatchPolicy
{
    public function __construct(
        #[Autowire(env: 'int:APP_BATCH_SIZE')]
        private int $batchSize,
    ) {
    }
}
```

Le typage ne remplace pas une règle métier : un entier négatif peut être techniquement convertible mais interdit comme taille de lot. Une valeur absente ou invalide peut échouer lors de la résolution du service ; ne présume pas que toute erreur est détectée à la compilation.

## 4. Configuration effective et environnements

Un bloc `when@test` exprime une différence de configuration pour les tests. `debug:config framework` montre la configuration fusionnée de l'extension ; `config:dump-reference framework` montre les possibilités et valeurs de référence. Ce ne sont pas deux vues du même objet.

`debug:container` inspecte les définitions et paramètres ; `debug:autowiring` aide à choisir un type injectable. `lint:yaml` contrôle la syntaxe des fichiers ; `lint:container` contrôle d'autres propriétés, notamment les types de dépendances. Aucun de ces contrôles ne démontre à lui seul un parcours fonctionnel correct.

## 5. Secrets et valeurs publiques

Les exemples de `.env` restent des valeurs par défaut sans secret. Les vraies valeurs locales vont dans `.env.local`, ignoré par Git, ou dans le coffre Symfony ; l'application y accède via `%env(...)%` ou l'attribut d'injection correspondant.

La clé privée de déchiffrement du coffre de production doit rester hors du dépôt. La possibilité de stocker les fichiers chiffrés ne rend pas acceptable de versionner la clé qui les déchiffre. Dans les exercices, utilise uniquement des valeurs fictives.

Les commandes de diagnostic de variables peuvent afficher leurs valeurs. Pour un compte rendu, relève le nom, la provenance et le type d'une variable fictive, pas les secrets du projet. [Documentation du coffre](https://symfony.com/doc/current/configuration/secrets.html).

## 6. Préparer la production

Une préparation reproductible part de `composer.lock` avec `composer install`, et non d'une mise à jour libre des dépendances. Les dépendances de développement ne sont pas nécessaires à l'exécution habituelle en production ; vérifie néanmoins que ton code ne les utilise pas.

Pour le laboratoire, effectue l'essai de démarrage prod dans une copie jetable. Un contrôle utile associe installation, lint du container, préchauffage du cache et requête sur une route connue. Tester seulement `cache:warmup` ne prouve pas qu'une action répond correctement.

Une variable d'environnement ne désactive pas automatiquement tous les listeners d'un bundle ; vérifie les conditions d'enregistrement. Après modification de classes ou de configuration, redémarre les workers de longue durée lorsque le déploiement l'exige.

## Observer dans l'application

```bash
php bin/console about
php bin/console debug:config framework --env=test
php bin/console config:dump-reference framework
php bin/console lint:yaml config/
php bin/console lint:container --env=test
php bin/console help debug:dotenv
```

Utilise `symfony console` si la CLI Symfony est disponible. `debug:dotenv` s'utilise sur les seules variables fictives des exercices ; consulte son aide pour le filtrage.

## Sources et différences de version

- [Configuration Symfony](https://symfony.com/doc/current/configuration.html).
- Sources installées : `vendor/symfony/dotenv/Dotenv.php` (`loadEnv`, `bootEnv`), `vendor/symfony/dependency-injection/EnvVarProcessor.php`.
- `src/Kernel.php` : la restriction d'environnements observée appartient à ce dépôt. Le cours Kernel identifie les ajouts 8.1 ; ne les généralise pas à toute application 8.0.
