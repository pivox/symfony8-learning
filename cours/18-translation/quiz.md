# Quiz — Module 18 : Translation

[Cours](README.md) · [Exercices](exercices.md)

Questions d'entraînement originales, socle Symfony 8.0.

**Q1 (une réponse).** Quel contrat injecter ?
A. `SerializerInterface` B. `TranslatorInterface` des Contracts C. `Request` D. `Filesystem`

**Q2 (une réponse).** Quel est le domaine habituel par défaut ?
A. `forms` B. `validators` C. `messages` D. `routes`

**Q3 (une réponse).** Une valeur utilisateur doit être :
A. Un paramètre du message B. Une partie variable de sa clé C. Un nom de domaine D. Un nom de fichier

**Q4 (une réponse).** `_locale` sur une route permet :
A. De traduire la base automatiquement B. D'envoyer un email C. De créer les catalogues D. De porter la locale dans la requête

**Q5 (une réponse).** Une clé absente de tous les catalogues peut donner :
A. Toujours une exception B. L'identifiant lui-même C. Un 404 systématique D. Une chaîne interdite par Twig

**Q6 (une réponse).** Un repli sert à :
A. Désactiver la traduction B. Traduire automatiquement avec une IA C. Chercher un message dans une autre locale D. Changer toutes les URL

**Q7 (une réponse).** Quel fichier convient aux messages ICU français du domaine messages ?
A. `messages+intl-icu.fr.yaml` B. `fr.messages.json.php` C. `validators.en.yaml` D. `.env.fr`

**Q8 (une réponse).** Pour un paramètre ICU nommé `count`, on passe normalement :
A. `['%count%' => 2]` B. `['#' => 2]` C. Aucun paramètre D. `['count' => 2]`

**Q9 (une réponse).** La branche ICU `=0` :
A. Correspond à tout pluriel B. Vise exactement zéro C. Rend le message anglais D. Efface le catalogue

**Q10 (une réponse).** Les erreurs des contraintes utilisent habituellement :
A. `routes` B. `console` C. `validators` D. `assets`

**Q11 (une réponse).** Le filtre Twig `trans` suivi du rendu habituel :
A. Reste soumis à l'échappement de sortie B. Rend toute donnée HTML sûre C. Supprime les scripts du paramètre D. Chiffre le message

**Q12 (une réponse).** Une locale explicite dans un appel à `trans()` :
A. Change définitivement tout le Kernel B. Change le navigateur C. Modifie les fichiers D. Sélectionne la langue de cet appel

**Q13 (une réponse).** L'extraction statique trouve-t-elle toute clé dynamique ?
A. Oui B. Non C. Oui si elle contient un point D. Oui avec une session

**Q14 (une réponse).** `TranslatableMessage` transporte principalement :
A. Une connexion réseau B. Une réponse HTTP C. Un identifiant avec ses paramètres et son domaine D. Un catalogue complet

**Q15 (plusieurs réponses).** Quels contrôles sont complémentaires ?
A. Syntaxe YAML B. Utiliser `raw` partout C. Ne tester que la clé brute D. Formater les messages avec leurs paramètres réels

## Réponses

1. **B** — Le traducteur résout les messages ; sérialisation, requête et système de fichiers ont d'autres rôles.
2. **C** — `forms` peut être choisi ; `validators` sert aux contraintes ; `routes` n'est pas le défaut.
3. **A** — Des clés dynamiques multiplient inutilement les ressources et compliquent l'extraction ; domaines et fichiers ne représentent pas chaque utilisateur.
4. **D** — La route détermine un contexte de langue, pas le contenu des traductions ni une opération externe.
5. **B** — Il faut donc vérifier les textes attendus ; absence d'exception ne signifie pas présence d'une traduction.
6. **C** — Un repli cherche une traduction déjà définie, sans génération de texte ni modification générale des routes.
7. **A** — Le suffixe active le format ICU ; les autres noms ne décrivent pas ce catalogue français.
8. **D** — ICU utilise ici `count`, tandis que `%count%` correspond à la convention des messages classiques.
9. **B** — Une valeur exacte permet de préciser le cas zéro indépendamment des catégories de pluriel de la locale.
10. **C** — Traduire les labels de formulaire ne suffit pas à traduire les violations du Validator.
11. **A** — Échapper n'est ni purifier un contenu HTML, ni chiffrer. `raw` retirerait cette protection de sortie.
12. **D** — L'appel ne réécrit ni le navigateur, ni les catalogues, ni l'état global du Kernel.
13. **B** — Un outil statique ne peut pas nécessairement calculer les clés construites à l'exécution ; points et sessions n'y changent rien.
14. **C** — Sa résolution peut être différée jusqu'au rendu ; ce n'est pas un catalogue ni une connexion.
15. **A, D** — Une syntaxe YAML valide n'assure pas le formatage ICU ; B fragilise l'échappement et C ne vérifie aucune traduction.
