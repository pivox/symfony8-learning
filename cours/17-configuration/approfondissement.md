# Approfondissement — transformer et résoudre une configuration

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Le cours explique la provenance des variables ; ce complément suit leur transformation et leur durée de vie.
Une valeur choisie correctement dans Dotenv peut encore être mal convertie ou déjà résolue dans un processus.
DependencyInjection et Dotenv sont installés ; toutes les variables de laboratoire sont fictives et préfixées `TUTO_`.
Les processeurs étudiés appartiennent au socle Symfony 8.0, disponible dans le dépôt 8.1.8.
Aucun exercice ne demande de modifier un secret ni le fichier `.env.local` personnel.

## 1. L'ordre des processeurs fait partie du contrat

Une expression `%env(int:TUTO_LIMIT)%` demande une conversion avant l'injection dans le service.
Plusieurs processeurs peuvent être composés : le processeur extérieur reçoit le résultat de ceux situés à sa droite.
`int:default:tuto.limit:TUTO_LIMIT` choisit donc d'abord un repli éventuel, puis convertit le résultat en entier.
Le nom `tuto.limit` désigne ici un paramètre du conteneur, pas une seconde variable d'environnement.
Le processeur `default:` accepte un repli quand la variable est absente ou renvoie une chaîne vide ou `null`.
Les valeurs `'0'` et `false` ne sont pas assimilées à une chaîne vide par ce mécanisme.
Une valeur présente mais incorrecte ne doit pas devenir silencieusement une valeur par défaut.

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;

// Fragment de paramètre ; tuto.default_delay doit être un paramètre défini.
#[Autowire(env: 'int:default:tuto.default_delay:TUTO_DELAY')]
int $delay,
```

Si `TUTO_DELAY=abc`, la conversion en entier échoue au lieu de sélectionner arbitrairement le repli.
Si la valeur est `'0'`, la conversion produit zéro ; une éventuelle interdiction relève ensuite du service.
Inverser des processeurs peut déplacer l'erreur avant que le repli ait une chance de s'appliquer.
Lis la chaîne comme des fonctions imbriquées, pas comme une simple liste de mots.
La [documentation des processeurs 8.0](https://symfony.com/doc/8.0/configuration/env_var_processors.html) décrit chaque transformation.

## 2. Un JSON valide n'est pas une politique valide

Le processeur `json:` est utile pour transmettre une petite structure de configuration.
Il vérifie le décodage, mais ne connaît ni les langues autorisées ni la signification de chaque champ.
Le service doit encore vérifier les types, la forme attendue et ses contraintes métier.
Une liste vide, une liste contenant des entiers et un objet associatif ne sont pas interchangeables.
Choisis un paramètre de repli du bon type pour l'endroit où il entre dans la chaîne.
Par exemple, avec `json:default:...`, le repli utilisé par `default:` doit encore être du JSON à décoder.

```text
Variable brute : ["compact","detailed"]
Conversion json : tableau de deux chaînes
Validation métier : valeurs autorisées, ordre et doublons maîtrisés
```

La première ligne peut être valide alors que la troisième échoue.
La conversion et la règle métier doivent donc avoir des cas négatifs séparés dans les tests.
Un entier négatif n'est pas une erreur de syntaxe YAML ni forcément une erreur du processeur `int:`.
Une liste JSON malformée n'est pas une simple liste métier vide.
Le message diagnostique doit garder cette distinction, sans exposer d'autres valeurs du processus.

| Entrée de la chaîne `int:default:...` | Résolution attendue | Contrôle suivant |
|---|---|---|
| Variable absente | Repli puis conversion | Règle métier |
| Chaîne vide | Repli puis conversion | Règle métier |
| `'0'` | Entier zéro | Autorisé ou refusé par le service |
| `'25'` | Entier 25 | Plage métier |
| `'abc'` | Erreur de conversion | Corriger la configuration |

## 3. Une résolution n'est pas une lecture continue du système

Le conteneur conserve les valeurs d'environnement déjà résolues dans un cache interne.
Un service partagé garde aussi les arguments reçus lors de sa construction.
Changer `$_ENV` après avoir récupéré le service ne réinjecte pas ses paramètres.
Même un second service résolu plus tard peut recevoir la valeur déjà mémorisée pour la même expression `%env(...)%`.
Cette distinction devient visible dans un worker qui reste démarré pendant longtemps.
Changer la configuration du déploiement ne transforme pas spontanément les objets du processus déjà actif.
Un redémarrage du processus est une frontière claire pour repartir avec un environnement actualisé.

```text
Résolution A avec TUTO_WINDOW=10 → valeur 10 mémorisée
Modification de la variable dans le même processus → objets existants inchangés
Nouvelle instance de conteneur avec TUTO_WINDOW=20 → nouvelle résolution possible
```

Le test du laboratoire contrôle explicitement `$_ENV` et `$_SERVER` pour une variable fictive unique.
Il restaure ensuite la présence ou l'absence initiale de cette variable dans les deux tableaux.
Un redémarrage du kernel ne supprime pas les variables PHP globales que ton propre test a modifiées.
La séparation entre cache du conteneur et environnement du processus doit donc apparaître dans le nettoyage.
Ne modifie pas une propriété interne du conteneur pour simuler un redémarrage.
Relis `Container::getEnv()` et sa propriété `envCache` dans le code installé pour expliquer l'observation.

## Observer dans l'application

Après création des services fictifs de l'atelier :

```bash
php bin/console help debug:dotenv
php bin/console debug:container --parameter=tuto.default_delay --env=test
php bin/console lint:container --env=test
php bin/phpunit tests/Configuration
```

La commande de paramètre suppose que tu as défini ce paramètre pour le fragment étudié.
Utilise un filtre sur les variables fictives si tu emploies `debug:dotenv` après lecture de son aide.
Une compilation réussie ne garantit pas que toutes les variables différées ont déjà été résolues.
Ajoute donc un appel réel au service qui consomme la variable, puis vérifie son résultat.
Sources locales : `vendor/symfony/dependency-injection/EnvVarProcessor.php` et `Container.php`.
La [configuration Symfony 8.0](https://symfony.com/doc/8.0/configuration.html#configuration-based-on-environment-variables) précise le rôle des expressions `%env(...)%`.
Les [exercices 9 et 10](exercices.md) isolent composition des conversions et durée de vie de la résolution.
