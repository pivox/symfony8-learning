# Quiz — Module 17 : Configuration

[Cours](README.md) · [Exercices](exercices.md)

Questions d'entraînement originales. Q13 porte sur le dépôt 8.1 ; les autres sur les mécanismes du socle 8.0.

**Q1 (une réponse).** Où trouver les versions résolues ?
A. `symfony.lock` B. `.env` C. `composer.lock` D. `config/bundles.php`

**Q2 (une réponse).** `symfony.lock` décrit principalement :
A. Les recettes Flex appliquées B. Les mots de passe C. Les sessions D. Les tests réussis

**Q3 (une réponse).** Par défaut, une vraie variable du processus :
A. Est toujours écrasée par `.env` B. Est ignorée en prod C. Doit être un entier D. Garde priorité sur les fichiers Dotenv

**Q4 (une réponse).** Quel fichier est ignoré par défaut en environnement test ?
A. `.env` B. `.env.local` C. `.env.test` D. `.env.test.local`

**Q5 (une réponse).** Entre `.env.dev` et `.env.dev.local`, lequel a priorité ?
A. Le plus ancien B. `.env.dev` C. `.env.dev.local` D. Aucun fichier n'est chargé

**Q6 (une réponse).** `APP_ENV` et `APP_DEBUG` :
A. Ont des rôles distincts B. Sont des synonymes C. Désignent deux bundles D. Doivent être secrets

**Q7 (une réponse).** Le processeur `int:` garantit-il une valeur métier positive ?
A. Oui B. Seulement en prod C. Seulement avec YAML D. Non

**Q8 (une réponse).** `debug:config framework` montre :
A. Tous les tests B. La configuration effective de l'extension C. La documentation complète D. Uniquement `.env`

**Q9 (une réponse).** `config:dump-reference` aide à connaître :
A. Le dernier statut HTTP B. Les secrets réels C. La structure et les valeurs de référence D. L'état des workers

**Q10 (une réponse).** Quelle commande respecte les versions verrouillées ?
A. `composer install` B. `composer update` sans contrainte supplémentaire C. `make:controller` D. `cache:clear`

**Q11 (une réponse).** Une clé privée de déchiffrement de production doit :
A. Être dans le README B. Être jointe aux fichiers chiffrés publics C. Servir de nom de branche D. Rester hors du dépôt

**Q12 (une réponse).** `lint:yaml` valide-t-il à lui seul une fonctionnalité ?
A. Oui B. Non C. Oui si tous les fichiers sont YAML D. Oui sans tests

**Q13 (une réponse, dépôt 8.1).** `APP_ENV=staging` dans le Kernel actuel :
A. Est forcément équivalent à prod B. Crée un environnement accepté automatiquement C. Est refusé par la restriction locale D. Active Doctrine

**Q14 (une réponse).** Un `.env.local.php` peut :
A. Modifier la source effective du chargement de `bootEnv()` B. Remplacer PHPUnit C. Supprimer tous les secrets D. Générer les routes

**Q15 (plusieurs réponses).** Pour un diagnostic partageable :
A. Copier tous les secrets B. Masquer toute provenance C. Indiquer la provenance d'une variable fictive D. Donner la commande et le message d'erreur expurgé si nécessaire

## Réponses

1. **C** — Les contraintes sont dans `composer.json`, les versions résolues dans son lock. Les autres fichiers ont d'autres responsabilités.
2. **A** — Flex suit ses recettes ; il ne suit ni authentification, sessions ni résultats de tests.
3. **D** — Sauf surcharge explicitement demandée au chargeur. Type, environnement prod et fichier `.env` ne changent pas cette règle par défaut.
4. **B** — `.env.test.local` peut être chargé : l'ignorer aussi serait une généralisation incorrecte.
5. **C** — La surcharge locale de l'environnement est plus spécifique ; l'ancienneté des fichiers ne décide pas.
6. **A** — L'un choisit l'environnement, l'autre le debug. Leur relation par défaut ne les rend pas synonymes ni secrets.
7. **D** — Un entier négatif reste un entier. Les règles métier doivent compléter la conversion.
8. **B** — La commande montre la configuration fusionnée, pas le catalogue complet des options ni les tests.
9. **C** — La référence décrit les possibilités ; l'exécution réelle se diagnostique avec d'autres commandes.
10. **A** — `update` résout de nouvelles versions ; les deux autres commandes ne gèrent pas les dépendances.
11. **D** — Publier la clé permettant de déchiffrer annule l'intérêt de la confidentialité des secrets chiffrés.
12. **B** — Syntaxe, injection et comportement sont des contrôles complémentaires.
13. **C** — Le Kernel du dépôt limite les environnements à dev/test/prod ; aucune création automatique ni activation Doctrine.
14. **A** — Un dump peut être chargé à la place des fichiers texte ; il n'a pas les fonctions proposées en B, C et D.
15. **C, D** — Une preuve peut rester reproductible sans diffuser de secrets. A expose les valeurs, B empêche de comprendre la cause.
