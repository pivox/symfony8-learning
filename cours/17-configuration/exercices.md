# Exercices — Module 17

[Cours](README.md) · [Quiz](quiz.md) · [Méthode](../README.md)

Code éventuel dans `src/Configuration/`, tests dans `tests/Configuration/`. Utilise exclusivement des variables fictives préfixées `TUTO_`. Les laboratoires Dotenv utilisent un répertoire temporaire et restaurent `$_ENV` et `$_SERVER` ; aucune modification de vrais secrets.

## 1. Guidé — établir la provenance

Choisis FrameworkBundle, Messenger et un paquet de développement. Dresse pour chacun contrainte, version verrouillée, recette éventuelle et fichier de configuration.

- **Critères :** chaque information renvoie au bon fichier ; distingue un paquet installé d'une recette absente et d'une fonctionnalité non configurée.
- **Indice :** `symfony.lock` et `composer.lock` répondent à des questions différentes.

## 2. Guidé — precedence dans un laboratoire

Crée quatre fichiers Dotenv temporaires donnant à `TUTO_LABEL` les valeurs `base`, `local`, `dev` et `dev-local`. Charge-les avec `Dotenv::loadEnv()` en environnement dev, sans variable système préexistante.

- **Critères :** résultat `dev-local` ; retire successivement les fichiers les plus prioritaires et retrouve `dev`, `local`, puis `base`. Chaque cas repart d'un environnement propre.
- **Indice :** des variables conservées par le test précédent faussent l'expérience.

## 3. Autonome — priorité du processus et test

Ajoute les variantes `.env.test` et `.env.test.local`. Compare chargement dev, chargement test et présence d'une variable `TUTO_LABEL=process` dans le processus.

- **Critères :** `.env.local` est ignoré en test ; `.env.test.local` peut gagner sur `.env.test` ; la variable du processus gagne lorsque la surcharge est désactivée.
- **Indice :** explicite les options du chargeur et restaure aussi les valeurs absentes initialement.

## 4. Autonome — injection et contraintes métier

Crée `BatchPolicy` injectant `TUTO_BATCH_SIZE` via le processeur `int:` ; la plage métier admise est de 1 à 100.

- **Critères :** 25 est accepté ; 0 et 101 sont refusés par la règle métier ; `abc` n'est pas considéré comme une taille valide. Teste la récupération du service par le container.
- **Indice :** identifie séparément erreur de conversion et violation de plage.

## 5. Diagnostic — configuration fusionnée

Dans une branche d'exercice, configure une taille par défaut à 20 et une valeur de test à 2, au moyen de paramètres et d'une surcharge `when@test`.

- **Critères :** deux commandes lancées dans des processus distincts montrent dev=20 et test=2 ; le service reçoit la valeur correspondante. Compare `debug:config`, `debug:container --parameter=...` et `config:dump-reference` sans leur attribuer le même rôle.
- **Indice :** une valeur dans le fichier ne prouve pas que ce fichier a la priorité effective.

## 6. Diagnostic — environnement invalide

Observe le démarrage avec `APP_ENV=staging` dans un sous-processus de test, puis avec `test`. N'ajoute pas `staging` au Kernel pour contourner le résultat.

- **Critères :** code de sortie non nul et message explicite pour `staging` dans ce dépôt ; démarrage réussi pour `test`. Explique la restriction depuis `src/Kernel.php`.
- **Indice :** c'est une politique de ce projet et une API du Kernel installé, pas une règle universelle de Symfony 8.0.

## 7. Autonome — secrets fictifs et diagnostic

Dans une copie jetable du projet, crée un secret de test `TUTO_API_TOKEN` avec la valeur fictive `training-only`, puis consomme-le via une dépendance injectée.

- **Critères :** le service reçoit la valeur ; aucun vrai identifiant n'est utilisé ; le compte rendu indique les fichiers produits, leur rôle et lesquels doivent rester privés. N'affiche pas la valeur dans les logs du service.
- **Indice :** consulter l'aide `secrets:*` avant d'exécuter les commandes ; les clés de déchiffrement et les données chiffrées n'ont pas le même rôle.

## 8. Défi final — diagnostic de démarrage

Dans une copie jetable, introduis successivement une valeur obligatoire absente, un entier invalide et une erreur de syntaxe YAML. Produis pour chaque cas la commande minimale qui révèle la cause, puis vérifie le démarrage corrigé en test et prod.

- **Critères :** trois erreurs distinguées et corrigées ; tests existants exécutés en test ; container et cache préparés en prod ; une route d'exercice connue donne la réponse attendue. Aucune dépendance n'est mise à jour avec `composer update`.
- **Livrable :** tableau « symptôme → preuve → correction → vérification » sans secrets.
- **Indice :** une erreur de syntaxe, de résolution de service et de comportement HTTP n'apparaît pas nécessairement au même stade.
