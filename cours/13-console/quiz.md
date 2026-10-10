# Quiz — Module 13 : Console

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md)

Socle 8.0 ; Q15 distingue les ajouts 8.1. Le nombre de réponses est précisé à chaque question.

**Q1 (une réponse).** Le rôle principal d'une commande mince est :
A. Remplacer le conteneur · B. Adapter des entrées CLI vers un service et présenter son résultat · C. Contenir toutes les règles métier · D. Créer une transaction Doctrine implicite

**Q2 (plusieurs réponses).** Sur Symfony 8.0 :
A. `#[AsCommand]` peut nommer une classe commande · B. Une commande invocable est possible · C. `#[Argument]` et `#[Option]` existent · D. Les commandes doivent toutes étendre Command

**Q3 (une réponse).** Une option VALUE_REQUIRED :
A. Est toujours obligatoire · B. Attend une valeur lorsqu'elle est présente · C. Refuse toute valeur · D. Est forcément un booléen

**Q4 (une réponse).** Pour terminer la lecture des options avant un argument commençant par `-` :
A. `//` · B. `--` · C. `::` · D. `#`

**Q5 (plusieurs réponses).** Parmi les statuts conventionnels :
A. SUCCESS vaut 0 · B. FAILURE vaut 1 · C. INVALID vaut 2 · D. Tout message d'erreur impose automatiquement FAILURE

**Q6 (une réponse).** `--dry-run` :
A. Annule automatiquement toute mutation PHP · B. Est un contrat à implémenter explicitement · C. Ajoute une base temporaire · D. Garantit le statut 0

**Q7 (une réponse).** Un rapport machine fiable doit :
A. Mélanger JSON et barre de progression · B. Changer selon la largeur du terminal · C. Posséder une structure stable séparée des ornements humains · D. Ignorer les lignes rejetées

**Q8 (plusieurs réponses).** En exécution non interactive :
A. `-n` désactive le dialogue · B. Les entrées essentielles doivent rester gérées · C. Un job ne doit pas attendre un terminal · D. Toute validation est désactivée

**Q9 (une réponse).** Ordre d'une commande classique :
A. execute, interact, initialize · B. initialize, interact si autorisé, execute après validation des entrées · C. interact seulement · D. ordre aléatoire

**Q10 (une réponse).** `CommandTester` déclenche-t-il les événements de l'application Console ?
A. Toujours · B. Seulement en mode verbose · C. Non, utiliser ApplicationTester pour cette frontière · D. Seulement si la commande retourne 0

**Q11 (plusieurs réponses).** Avec un tester, on peut vérifier :
A. Le code de sortie · B. La sortie affichée · C. Les réponses interactives simulées · D. Le comportement sans aucune assertion

**Q12 (une réponse).** Un listener de terminaison :
A. Peut modifier le code de sortie · B. Est une transaction universelle · C. Garantit son exécution après un arrêt brutal · D. Remplace automatiquement la commande

**Q13 (une réponse).** `--silent` :
A. Transforme les échecs en succès · B. Supprime les sorties, y compris les erreurs affichées, sans effacer le statut · C. Écrit forcément un rapport JSON · D. Interdit la lecture de fichiers

**Q14 (une réponse).** En Symfony 8.0, lors d'un assemblage manuel, quelle méthode utiliser pour enregistrer une commande ?
A. Application::add() · B. Application::addCommand() · C. Application::route() · D. Application::listen()

**Q15 (plusieurs réponses, version 8.1 hors certification).** Quelles nouveautés sont propres à 8.1 ?
A. AsCommand sur des méthodes · B. CommandTester::run() et ExecutionResult · C. Les premiers attributs Argument et Option · D. ConsoleBundle

## Corrections raisonnées

1. **B.** La commande adapte entrées et résultat, le service porte le traitement. A nie la DI, C empêche la séparation des responsabilités et D suppose une dépendance et un comportement absents.
2. **A, B, C.** Les invocables et attributs d'entrée existent depuis 7.3. D est faux : l'adaptateur Symfony permet précisément une classe invocable sans cet héritage.
3. **B.** La présence est facultative, sa valeur ne l'est pas lorsqu'elle est utilisée. A confond deux exigences ; C décrit VALUE_NONE ; D ne correspond pas à une valeur obligatoire ordinaire.
4. **B.** `--` marque la fin des options pour le parseur. A, C et D n'ont pas ce rôle et peuvent devenir des données ou être interprétés par le shell.
5. **A, B, C.** Ce sont les trois constantes usuelles. D est faux : écrire une erreur et retourner zéro reste possible, d'où l'intérêt des tests de statut.
6. **B.** Il faut décider ce qui est observé et ce qui n'est pas écrit. A et C inventent une infrastructure ; D est faux, des données invalides restent invalides pendant une simulation.
7. **C.** Un consommateur machine dépend d'un format prévisible. A casse le parseur, B rend le format instable et D supprime des informations essentielles au diagnostic.
8. **A, B, C.** Le contrat doit fonctionner sans conversation bloquante. D est faux : supprimer une interaction ne supprime pas le parsing ni les contrôles métier.
9. **B.** L'initialisation précède le dialogue éventuel et l'exécution. A inverse le cycle ; C omet le traitement ; D empêcherait tout contrat reproductible.
10. **C.** Le tester direct n'exécute pas l'orchestration événementielle de l'application. A, B et D inventent une dépendance à la verbosité ou au succès qui n'existe pas.
11. **A, B, C.** Statut, affichage et entrées simulées sont observables. D est faux : l'exécution seule ne prouve pas que le comportement attendu a eu lieu.
12. **A.** L'événement permet d'ajuster le statut. B et C promettent une garantie de transaction/arrêt absente ; D ne décrit pas le rôle d'une terminaison.
13. **B.** Le silence porte sur les flux, pas sur la réussite. A confond ces deux dimensions ; C et D ne découlent pas de cette option globale.
14. **B.** addCommand() remplace add(), retirée en 8.0 après dépréciation en 7.4. A est donc obsolète ; C et D ne sont pas les API d'enregistrement de commandes.
15. **A, B, D.** Ces ajouts figurent dans le changelog 8.1. C est faux : Argument et Option sont apparus en 7.3, donc appartiennent déjà au socle de l'examen 8.0.
