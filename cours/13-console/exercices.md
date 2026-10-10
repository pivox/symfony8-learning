# Exercices — Module 13 : import simulé et rapport d'erreurs

[Parcours](../README.md) · [Cours](README.md) · [Quiz](quiz.md)

Code : `src/Console/`, tests : `tests/Console/`, exécution : `php bin/phpunit tests/Console`.
Le CSV possède l'en-tête `email,name`. L'import produit un résultat en mémoire ; aucune persistance ni notification réelle.
Les rapports vont uniquement dans un répertoire temporaire propre au test ; chaque test nettoie ses fichiers.
Services et commandes : `KernelTestCase`. Fixtures : ligne 2 valide `ada@example.test,Ada`, ligne 3 invalide `incorrect,Bob`.

## 1. Exposer une commande identifiable — guidé

Scénario : `app:contacts:inspect` reçoit le chemin d'un fichier CSV en argument obligatoire et affiche le chemin retenu.
Travail : classe service avec `AsCommand`, aide claire et argument explicite ; utiliser les attributs invocables ou l'API classique.

- Attendus : `list app` montre la commande ; `--help` décrit argument et rôle de l'outil ; invocation valide retourne 0.
- Acceptation : `CommandTester` trouve la commande via l'application et vérifie la sortie contenant le chemin fourni.
- Cas limite : argument absent sans interaction → échec ; distinguer exception du tester direct et sortie de l'application.
- Indice : le nom de service PHP n'est pas le nom de commande destiné au shell.

## 2. Construire le résultat d'analyse — guidé

Scénario : le service lit les lignes avec les fonctions CSV PHP et retourne un résultat structuré.
Travail : déléguer la lecture et la validation au service, afficher seulement sa synthèse dans la commande.

- Attendus : fixture mixte → 2 lignes lues, 1 acceptée, 1 rejetée ; erreur d'email rattachée à la ligne 3.
- Acceptation : CSV avec uniquement l'en-tête → compteurs à zéro et succès ; CSV vide sans en-tête → structure invalide.
- Cas limite : noms contenant une virgule entre guillemets correctement lus ; ne pas découper avec `explode(',')`.
- Indice : préciser les arguments CSV pertinents de PHP 8.4 et vérifier les signatures locales.

## 3. Définir une simulation observable — semi-guidé

Scénario : `--report=chemin` écrit un rapport ; `--dry-run` doit analyser toutes les données mais ne créer aucun rapport.
Travail : ajouter les options et définir le contrat du dry-run dans l'aide.

- Attendus : avec `--report`, rapport créé ; avec les deux options, mêmes compteurs mais aucun fichier créé.
- Acceptation : fichier déjà présent à l'emplacement visé reste inchangé en dry-run ; statut dépend toujours de la validité des lignes.
- Cas limite : absence de `--report` n'exige pas de valeur ; présence de `--report` sans valeur est une erreur d'entrée.
- Indice : « option à valeur requise » ne signifie pas « option obligatoire ».

## 4. Rendre l'échec automatisable — semi-guidé

Scénario : le pipeline doit distinguer réussite, rejet de lignes et incohérence de paramètres.
Travail : définir `--limit` positif (défaut 100), puis appliquer le contrat de statuts du cours.

- Attendus : toutes lignes valides → 0 ; au moins une rejetée → 1 ; `--limit=0` ou `-1` → 2.
- Acceptation : fichier introuvable/illisible → 1 avec message utile ; le résumé n'annonce jamais un succès trompeur.
- Cas limite : `--limit=1` traite une seule ligne de données et indique clairement cette limite dans le rapport.
- Indice : tester `getStatusCode()` et un élément de sortie ; les couleurs ne sont pas le contrat.

## 5. Séparer rapport machine et diagnostic humain — autonome

Scénario : un rapport JSON contient `read`, `accepted`, `rejected`, `errors`, avec numéros de lignes et chemins des champs.
Travail : produire ce format de manière déterministe et réserver les détails d'analyse au mode verbeux.

- Attendus : JSON désérialisable, compteurs exacts, erreur `email` sur la bonne ligne ; pas de bordure ASCII dans le fichier.
- Acceptation : même donnée → même structure ; mode silencieux garde le bon statut ; le rapport ne recopie pas de données inutiles.
- Cas limite : chemin de rapport non inscriptible → échec opérationnel explicite, sans faux succès.
- Indice : comparer le JSON comme structure ; comparer les sorties humaines par fragments métier.

## 6. Ajouter une interaction maîtrisée — autonome

Scénario : l'option `--confirm` demande l'accord avant l'analyse ; par défaut, la commande reste entièrement automatisable.
Travail : définir les comportements réponse oui, réponse non et `--no-interaction` avec `--confirm`.

- Attendus : oui → traitement normal ; non → arrêt explicite sans analyse ni rapport ; fixer ce refus à statut 1.
- Acceptation : `--confirm --no-interaction` → statut 2 et explication, sans blocage ; sans `--confirm`, mode non interactif fonctionne.
- Tests : `setInputs()` pour les deux réponses ; vérifier l'absence d'appel au service en cas de refus.
- Indice : ne pas confondre une valeur par défaut de question avec une autorisation explicite.

## 7. Comprendre un listener qui ne tourne pas — diagnostic

Scénario : un listener `console.terminate` compte les exécutions, mais un test avec `CommandTester` n'observe rien.
Travail : enregistrer le listener avec `AsEventListener` et comparer le tester de commande à `ApplicationTester`.

- Attendus : tester direct → aucun événement applicatif ; tester d'application → événement et code de sortie observables.
- Acceptation : listener trouvé par `debug:event-dispatcher console.terminate` ; test sans arrêt du processus PHPUnit.
- Cas limite : erreur d'exécution observée via `console.error` au niveau application ; ne pas inventer sa présence au niveau commande.
- Indice : désactiver l'auto-exit de l'application de test ; c'est l'application qui orchestre les événements.

## 8. Défi intégrateur — outil exploitable en CI

Scénario : un job doit analyser un CSV, garder un rapport précis et échouer en présence de lignes invalides.
Travail autonome : réunir aide, service, options, rapport, statuts et diagnostic, puis écrire une procédure de reproduction minimale.

- Attendus : fixture de 4 lignes de données, dont 2 invalides → résumé 4/2/2, erreurs sur les bonnes lignes, statut 1.
- Acceptation : même entrée en dry-run → même résumé/statut, aucun fichier créé ou modifié ; entrée entièrement valide → 0.
- Couvrir : limite incohérente, fichier absent, en-tête incorrect, CSV cité, refus interactif, exécution non interactive et listener applicatif.
- Livraison : tests reproductibles sans réseau ni base, fichiers temporaires nettoyés et aide permettant à un tiers de lancer la commande.
- Indice : une commande fiable définit ensemble son entrée, sa sortie et son statut ; aucun de ces trois éléments ne suffit seul.
