# Exercices — Module 14 : du contrat à la régression

[Parcours](../README.md) · [Cours](README.md) · [Quiz](quiz.md)

Code : `src/Testing/`, tests : `tests/Testing/`, exécution : `php bin/phpunit tests/Testing`.
Scénario public en mémoire : catalogue de trois livres, recherche et rapport Console ; aucun stockage durable ni authentification.
Fixtures : `Symfony pratique`, `Symfony avancé`, `HTTP expliqué`. Créer des composants minimaux si les modules précédents ne sont pas faits.
Les tests utilisent le niveau demandé ci-dessous. Les exercices n'exigent pas de solution complète préexistante.

## 1. Définir un objet valeur — guidé

Scénario : `SearchTerm` conserve une chaîne nettoyée aux bords, refuse une valeur vide et garde la casse originale.
Travail : écrire un test d'objet pur avec `TestCase` et un provider PHPUnit à jeux nommés.

- Attendus : ` Symfony ` → `Symfony` ; `HTTP` → `HTTP` ; `''` et espaces seuls → exception explicitement choisie.
- Acceptation : comparaison stricte du résultat et de l'exception ; chaque frontière apparaît comme un cas nommé.
- Cas limite : le provider ne démarre pas le kernel et ne lit pas l'heure ni le réseau.
- Indice : un provider fournit les entrées ; il ne doit pas déjà exécuter le comportement à vérifier.

## 2. Appeler le service réel — guidé

Scénario : `Catalogue::search()` recherche sans tenir compte de la casse dans les trois titres de fixture.
Travail : `KernelTestCase`, service du conteneur, assertions sur contenu et ordre documenté.

- Attendus : `symfony` retourne les deux titres Symfony ; `http` retourne un titre ; terme absent → tableau vide.
- Acceptation : les résultats conservent un identifiant et leur titre ; l'ordre est déterministe, défini par le scénario.
- Cas limite : deux appels successifs n'ajoutent ni ne retirent un livre à la fixture.
- Indice : un test qui vérifie seulement la classe du service ne prouve aucune règle de recherche.

## 3. Remplacer une frontière — semi-guidé

Scénario : le service consulte une interface de source de catalogue ; simuler une source indisponible puis une source vide.
Travail : remplacer l'identifiant injecté dans le conteneur de test avant de récupérer le service consommateur.

- Attendus : source vide → résultat vide ; indisponibilité → exception métier attendue, distincte d'une absence de résultat.
- Acceptation : aucun accès réseau ; l'argument de recherche transmis à la dépendance respecte le contrat fixé.
- Cas limite : un test suivant retrouve la source normale après redémarrage du kernel.
- Indice : vérifier l'alias réellement utilisé avec `debug:container 'App\Testing\Catalogue' --env=test` si le double n'est pas appelé ; la sortie du service affiche ses arguments.

## 4. Tester une frontière HTTP — semi-guidé

Scénario : `GET /lab/testing/books?q=symfony` retourne une réponse JSON ; une fiche inconnue retourne 404.
Travail : `WebTestCase`, route en attributs, contrôleur mince et service des exercices précédents.

- Attendus : recherche → 200, Content-Type JSON et deux éléments ; fiche inconnue → 404.
- Acceptation : recherche vide selon ton contrat → statut documenté et testé ; POST sur la route GET → 405.
- Cas limite : interdire un simple `assertResponseIsSuccessful()` sans assertion sur le corps et le type de contenu.
- Indice : appeler `createClient()` directement ; il démarre déjà le kernel.

## 5. Parcourir une petite page HTML — autonome

Scénario : une page de démonstration isolée affiche les trois titres et un lien vers chaque fiche, rendu avec Twig installé.
Travail : vérifier le titre, le nombre de liens et suivre un lien avec le Crawler ; générer les URL avec `path()`.

- Attendus : `h1` contient `Catalogue de test`, trois liens de livres, clic sur le premier → fiche du bon livre.
- Acceptation : assertion sur l'identité du livre après navigation ; aucune comparaison de tout le HTML ni dépendance aux espaces d'indentation.
- Cas limite : un sélecteur inexistant est identifié comme cause de l'échec, distinct d'un 500 de la page.
- Indice : le Crawler observe le HTML serveur ; il n'exécute pas le JavaScript.

## 6. Chasser une fuite d'état — autonome

Scénario : une classe conserve un compteur statique et un test laisse un fichier de rapport commun aux suivants.
Travail : reproduire la dépendance à l'ordre, puis rendre les fixtures, fichiers et services indépendants.

- Attendus : chaque test passe seul et dans la suite, y compris `--order-by=random` ; aucun fichier commun résiduel.
- Acceptation : répertoire temporaire unique nettoyé même en cas d'échec ; une requête HTTP n'hérite pas d'un double posé dans une autre.
- Cas limite : documenter l'effet du reboot du client et pourquoi `disableReboot()` ne supprime pas tous les resets de services.
- Indice : isoler l'état à la source avant de verrouiller artificiellement l'ordre des tests.

## 7. Remonter d'un 500 à sa cause — diagnostic

Scénario : une fiche censée retourner JSON renvoie une page 500 et l'assertion JSON masque l'exception initiale.
Travail : provoquer une exception contrôlée du service, isoler le test et utiliser `catchExceptions(false)` pour comprendre la pile.

- Attendus : diagnostic indiquant première cause, frontière concernée et test minimal ; correction accompagnée d'un test de régression.
- Acceptation : le test échoue pour la bonne raison avant correction, passe après, et la réponse conserve statut/format/contenu attendus.
- Extension facultative : installer `symfony/profiler-pack` via Composer/Flex, vérifier sa configuration en test, puis collecter un profil sur une seule requête.
- Cas facultatif : vérifier que le profil existe avant d'interroger un collecteur effectivement disponible ; aucun collecteur Doctrine supposé.
- Indice : lire les logs de l'environnement concerné ; une erreur de parsing client est souvent une conséquence.

## 8. Défi intégrateur — preuve complète d'une régression

Scénario : une modification de la normalisation du terme casse simultanément recherche HTTP et rapport Console.
Travail autonome : écrire le plus petit test rouge sur le service, corriger, puis couvrir les deux frontières utilisatrices.

- Attendus : recherche ` Symfony ` retourne deux titres ; la commande rapporte deux résultats et un statut 0.
- Acceptation : objet valeur testé avec `TestCase`, service avec `KernelTestCase`, HTTP avec `WebTestCase`, commande avec tester et kernel.
- Couvrir : entrée vide, aucun résultat, source défaillante, méthode HTTP interdite et sortie Console cohérente.
- Livraison : note de diagnostic avant/après, commande de reproduction et suite entière verte indépendamment de l'ordre.
- Indice : un test de service localise la règle ; les tests de frontière prouvent que l'application utilise réellement cette règle.
