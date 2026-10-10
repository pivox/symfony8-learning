# Quiz — Module 11 : Routing

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md)

Socle Symfony 8.0. Chaque question indique le nombre de réponses ; corrections raisonnées après les 15 questions.

**Q1 (une réponse).** Quelle donnée ne participe pas à la comparaison ordinaire du chemin ?
A. Le segment `/books` · B. Le placeholder `{slug}` · C. La query `?q=php` · D. Le requirement de `slug`

**Q2 (plusieurs réponses).** Un requirement `year: '\d{4}'` :
A. Filtre les candidats au matching · B. Remplace toute validation métier · C. Peut être contrôlé à la génération · D. Prouve l'existence d'une archive

**Q3 (une réponse).** Une priorité d'attribut 20 face à une priorité 0 signifie :
A. Exécution après 0 · B. Reconnaissance avant 0 · C. Autorisation plus forte · D. Génération deux fois plus rapide

**Q4 (une réponse).** Sur une route GET connue, une requête POST sans autre route compatible produit normalement :
A. 200 · B. 301 obligatoirement · C. 405 · D. 422

**Q5 (plusieurs réponses).** Quels paramètres ont un rôle réservé ?
A. `_locale` · B. `_format` · C. `productCode` · D. `_controller`

**Q6 (une réponse).** `_format: 'json'` garantit :
A. Sérialisation de tout objet retourné · B. Valeur de format de la requête · C. Validation JSON complète · D. Authentification API

**Q7 (une réponse).** Le générateur reçoit d'abord :
A. Une classe de contrôleur · B. Un nom de méthode PHP · C. Un nom de route · D. Un code HTTP

**Q8 (plusieurs réponses).** Lors de la génération :
A. Un paramètre supplémentaire peut devenir une query · B. `_fragment` peut produire une ancre · C. Un nom inconnu est ignoré · D. Un paramètre obligatoire absent peut lever une exception

**Q9 (une réponse).** Le fragment `#details` d'une URL :
A. Arrive toujours dans `Request::query` · B. Est transmis dans le corps · C. N'est pas envoyé par le navigateur au serveur · D. Détermine la méthode HTTP

**Q10 (une réponse).** Pour générer le chemin localisé anglais d'un nom logique :
A. Changer seulement le texte de la page · B. Fournir `_locale: en` au générateur · C. Ajouter `Accept-Language` au tableau métier · D. Utiliser `priority: en`

**Q11 (plusieurs réponses).** Une route limitée à un hôte :
A. Exige de tester le host en HTTP · B. Peut utiliser des placeholders d'hôte · C. Ignore toujours son host à la génération · D. Fonctionne sans DNS réel dans KernelBrowser

**Q12 (une réponse).** Pour un lien complet émis en CLI, la question centrale est :
A. Le nombre de workers · B. Le contexte hôte/schéma du générateur · C. Le type de formulaire · D. Le mot de passe du visiteur

**Q13 (une réponse).** Deux routes de même nom :
A. Constituent automatiquement deux variantes de langue · B. Sont toujours départagées par le contrôleur · C. Risquent de se remplacer au chargement · D. Reçoivent automatiquement deux noms uniques

**Q14 (plusieurs réponses).** Pour diagnostiquer une fiche inconnue :
A. Utiliser `router:match` · B. Distinguer 404 de matching et 404 métier · C. Vérifier host et méthode · D. Supprimer le requirement sans preuve

**Q15 (une réponse).** Quel repère est correct pour Symfony 8.0 ?
A. Utiliser `Routing\Annotation\Route` · B. XML est le format recommandé · C. Utiliser `Routing\Attribute\Route` · D. `routing.controllers` exige 8.1

## Corrections raisonnées

1. **C.** La query ne participe pas au matching du chemin. A et B en définissent la structure ; D restreint une valeur de segment.
2. **A, C.** Les requirements filtrent la reconnaissance et sont vérifiés par le générateur selon son mode strict. B et D sont faux : la règle métier et l'existence restent à vérifier ailleurs.
3. **B.** Plus grand signifie plus tôt dans l'ordre des routes. A inverse l'ordre ; C confond Security et Routing ; D invente une propriété de performance.
4. **C.** Le chemin existe mais la méthode est interdite. A ignore le contrat, B n'est pas une obligation de redirection et D concerne plutôt une donnée invalide déjà reçue.
5. **A, B, D.** Ces noms pilotent locale, format et contrôleur. C est un nom métier ordinaire sans comportement spécial universel.
6. **B.** Le format est une information de requête. A nécessite un mécanisme de réponse/sérialisation, C un décodage et une validation, D une couche Security.
7. **C.** La route est référencée par son nom stable. A et B peuvent être des cibles du routage, pas son entrée de génération ; D est un résultat HTTP.
8. **A, B, D.** Query, fragment et paramètres obligatoires appartiennent au contrat du générateur. C est faux : un nom inconnu produit `RouteNotFoundException`.
9. **C.** Le navigateur garde le fragment côté client. A et B inventent un transport serveur ; D confond navigation locale et méthode HTTP.
10. **B.** Le générateur sélectionne la variante par locale. A ne change que le contenu ; C ne configure pas cette sélection ; D n'est pas une priorité valide.
11. **A, B, D.** L'hôte fait partie du contexte ; les paramètres serveur suffisent au client de test. C est faux : la génération doit aussi respecter l'hôte de la route.
12. **B.** En CLI, aucune requête visiteur ne fournit naturellement ce contexte. A n'explique pas un mauvais domaine ; C et D n'ont aucun rôle dans la génération du lien.
13. **C.** L'unicité est requise pour référencer sans ambiguïté les routes. A nécessite une déclaration localisée explicite ; B intervient trop tard ; D n'est pas un mécanisme garanti.
14. **A, B, C.** Ces vérifications séparent les couches et reproduisent la requête. D change le contrat sans identifier la cause et peut introduire une collision supplémentaire.
15. **C.** Les attributs utilisent ce namespace. A correspond aux anciens alias retirés ; B ignore le retrait XML en 8.0 ; D est faux, la ressource existe depuis 7.4.
