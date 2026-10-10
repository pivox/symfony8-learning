# Quiz — Module 12 : Validator

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md)

Socle 8.0, avec une question explicitement versionnée 8.1. Une ou plusieurs réponses selon l'indication.

**Q1 (une réponse).** `ValidatorInterface::validate()` retourne normalement :
A. Un booléen · B. Une liste de violations · C. Un DTO corrigé · D. Un statut HTTP

**Q2 (plusieurs réponses).** À propos de présence et format :
A. `NotNull` accepte `''` · B. `NotBlank` interdit normalement `''` · C. `Email` prouve la réception réelle d'un email · D. `Length(min: 2)` rejette `''`

**Q3 (une réponse).** Sans groupes explicites, une validation ordinaire utilise :
A. Tous les groupes · B. `Default` · C. Aucun groupe · D. Le groupe du Serializer

**Q4 (une réponse).** Une contrainte uniquement dans `publish`, lors d'une validation `Default` ordinaire :
A. S'exécute toujours · B. S'exécute si elle est la première · C. N'est pas sélectionnée · D. Est supprimée des métadonnées

**Q5 (plusieurs réponses).** `['Default', 'Strict']` :
A. Sélectionne deux groupes · B. Garantit que Strict attend le succès de Default · C. N'est pas une GroupSequence · D. Équivaut à un rôle Security

**Q6 (une réponse).** Avec `new GroupSequence(['Default', 'Strict'])`, si Default échoue :
A. Strict est ignoré · B. Strict est automatiquement corrigé · C. Le DTO est supprimé · D. Une exception est toujours lancée

**Q7 (une réponse).** Une séquence placée sur la classe `SignupInput` doit inclure :
A. `Default` à la place du nom de classe · B. Le groupe `SignupInput` · C. Le nom du contrôleur · D. La locale

**Q8 (plusieurs réponses).** `#[Valid]` sur une propriété objet :
A. Active la cascade · B. Interdit automatiquement null · C. Permet des chemins imbriqués · D. Rend l'objet persistant

**Q9 (une réponse).** Quelle association est correcte ?
A. `All` décrit uniquement le nombre d'éléments · B. `Count` valide le format des emails · C. `Collection` décrit les clés d'un tableau · D. `Valid` remplace tous les types PHP

**Q10 (une réponse).** Le chemin d'une violation désigne :
A. Toujours un fichier PHP · B. La position de la donnée concernée dans l'objet ou collection · C. La route HTTP · D. L'identifiant du validateur dans le conteneur

**Q11 (plusieurs réponses).** Une contrainte personnalisée réutilisable :
A. Peut être un attribut étendant Constraint · B. Associe une classe de validation · C. Doit contenir directement ses services · D. Peut utiliser un validateur autoconfiguré avec DI

**Q12 (une réponse).** Une erreur de cohérence de classe destinée au champ confirmation utilise :
A. `atPath('confirmation')` · B. Une route portant ce nom · C. `setStatusCode('confirmation')` · D. Un groupe Serializer

**Q13 (une réponse).** Le statut par défaut d'un échec de validation avec MapRequestPayload est :
A. 200 · B. 301 · C. 422 · D. 503

**Q14 (plusieurs réponses).** Pour un test robuste de violation :
A. Vérifier le chemin · B. Vérifier le code pertinent · C. Vérifier uniquement un message traduit · D. Vérifier les groupes réellement utilisés

**Q15 (une réponse, version 8.1 hors certification).** Quelle évolution est correcte ?
A. ConstraintValidator interdit désormais validate() · B. L'interface évolue vers validateInContext(), la classe de base adapte le contexte · C. Tous les validateurs doivent être appelés manuellement · D. Les contraintes deviennent des routes

## Corrections raisonnées

1. **B.** L'appel retourne une liste exploitable et comptable. A perd les détails, C confond normalisation et validation, D appartient à l'adaptateur HTTP.
2. **A, B, D.** NotNull vise null ; NotBlank vise notamment le vide ; Length compare bien une chaîne vide au minimum. C est faux : le format ne prouve ni existence ni réception.
3. **B.** Default est la sélection ordinaire, éventuellement redéfinie par une séquence de classe. A et C ne sont pas le comportement normal ; D mélange deux systèmes indépendants.
4. **C.** La contrainte existe mais n'appartient pas à la sélection. A et B inventent une exécution automatique ; D confond sélection et suppression de métadonnées.
5. **A, C.** Le tableau sélectionne sans condition de succès entre groupes. B décrit une séquence ; D n'a aucun rapport avec les groupes du Validator.
6. **A.** La séquence n'avance qu'après réussite du groupe précédent. B et C ajoutent des mutations absentes ; D confond invalidité métier et erreur technique.
7. **B.** Le nom court représente les contraintes ordinaires de la classe. A créerait une référence interdite à Default dans sa propre substitution ; C et D ne remplissent pas ce rôle.
8. **A, C.** La cascade donne accès aux règles et chemins enfants. B nécessite une contrainte de présence ; D relève d'une persistence qui n'est pas installée ici.
9. **C.** Collection contrôle une structure à clés. A est le rôle de Count ; B est celui d'une contrainte comme Email appliquée aux éléments ; D attribue à Valid un rôle de typage inexistant.
10. **B.** Par exemple `contacts[0].email`. A, C et D sont des identifiants techniques d'autres couches, pas le chemin de donnée d'une violation.
11. **A, B, D.** Métadonnées de contrainte et service d'exécution sont séparés. C est faux : les services injectés appartiennent au validateur, les métadonnées doivent rester sérialisables/cacheables.
12. **A.** Le builder rattache la violation à une propriété relative au contexte courant. B et C relèvent d'autres API ; D ne localise aucune erreur Validator.
13. **C.** Le resolver utilise 422 par défaut pour la validation. A annonce à tort le succès, B une redirection, D une indisponibilité ; le statut est configurable si le contrat le demande.
14. **A, B, D.** Ces éléments décrivent une règle et son exécution. C seul est fragile face aux traductions et ne prouve ni la bonne propriété ni le bon contexte.
15. **B.** L'adaptation est fournie par ConstraintValidator en 8.1. A est faux, sa méthode validate() reste à implémenter ; C contourne l'orchestration ; D est sans lien avec cette évolution.
