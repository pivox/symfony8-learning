# Quiz — Module 07 : Serializer avancé

Une ou plusieurs bonnes réponses : c'est indiqué. Réponses en bas.

**Q1 (une réponse).** Quel est le rôle d'un *normalizer* ?
A. Convertir un tableau en string JSON B. Convertir un objet en tableau (et inversement) C. Valider un objet D. Gérer les routes

**Q2 (plusieurs réponses).** Encoders fournis par le composant :
A. `JsonEncoder` B. `XmlEncoder` C. `CsvEncoder` D. `ProtobufEncoder`

**Q3 (une réponse).** Quel normalizer natif est interrogé en dernier (priorité -1000) ?
A. `DateTimeNormalizer` B. `PropertyNormalizer` C. `ObjectNormalizer` D. `JsonSerializableNormalizer`

**Q4 (une réponse).** Un normalizer custom sans tag/priorité explicite (autoconfiguration) a quelle priorité ?
A. -1000 B. 0 C. 1000 D. Aucune, il faut un tag manuel

**Q5 (une réponse).** Différence entre `ObjectNormalizer` et `PropertyNormalizer` ?
A. Aucune B. `ObjectNormalizer` utilise PropertyAccess (getters/setters/propriétés publiques), `PropertyNormalizer` accède aux propriétés par réflexion, y compris privées C. `PropertyNormalizer` est plus lent D. `ObjectNormalizer` ne gère pas les setters

**Q6 (une réponse).** Que retourne `getSupportedTypes(?string $format)` ?
A. La liste des formats B. Un tableau `type => bool|null` indiquant si le résultat de `supports*()` est cachable (null = non supporté) C. La priorité D. Les groupes

**Q7 (une réponse).** On a `#[Groups(['read'])]` sur certaines propriétés ; on sérialise **sans** option `groups`. Résultat ?
A. Seulement les propriétés sans groupe B. Toutes les propriétés accessibles C. Exception D. Rien

**Q8 (une réponse).** `#[MaxDepth(1)]` est ignoré si :
A. `groups` est absent B. `enable_max_depth` n'est pas à `true` dans le contexte C. L'objet est un tableau D. Le format est XML

**Q9 (une réponse).** Valeur par défaut de `circular_reference_limit` :
A. 0 B. 1 C. 5 D. illimitée

**Q10 (plusieurs réponses).** Comment gérer une référence circulaire ?
A. `circular_reference_handler` B. `#[MaxDepth]` + `enable_max_depth` C. Utiliser des groupes qui coupent la boucle D. Désactiver `ObjectNormalizer`

**Q11 (une réponse).** `#[SerializedName('mail')]` sur `$email` :
A. Renomme la propriété PHP B. La clé exposée dans la sortie/entrée est `mail` C. Chiffre le champ D. Ne fonctionne qu'en XML

**Q12 (une réponse).** Quel changement est intervenu en Symfony 8.0 pour `NameConverterInterface` ?
A. Interface supprimée B. `normalize()`/`denormalize()` reçoivent aussi `?string $class`, `?string $format`, `array $context` ; `AdvancedNameConverterInterface` supprimé C. Devenue `final` D. Fusionnée avec `NormalizerInterface`

**Q13 (plusieurs réponses).** Quels éléments sont supprimés/dépréciés autour de 8.0/8.1 ?
A. Le namespace `Symfony\Component\Serializer\Annotation` (supprimé en 8.0) B. `CsvEncoder::ESCAPE_CHAR_KEY` (supprimé en 8.0) C. `PartialDenormalizationException::getErrors()` (déprécié en 8.1) D. `#[Groups]` (supprimé)

**Q14 (une réponse).** Comment désérialiser un tableau d'objets ?
A. `deserialize($json, Author::class, 'json')` B. `deserialize($json, Author::class.'[]', 'json')` C. `deserializeAll(...)` D. Impossible

**Q15 (plusieurs réponses).** À quoi sert l'attribut `#[Serialize]` (HttpKernel 8.1) ?
A. Sérialiser automatiquement la valeur retournée par le contrôleur en réponse B. Accepter `code`, `headers`, `context` C. Désérialiser le corps de requête D. Remplacer `#[MapRequestPayload]`

---

## Questions de mise en situation — 16 à 20

**Q16.** (1 réponse) Un constructeur exige `int $count` sans défaut et la donnée omet `count`. En dénormalisation simple, on attend :

- A. Une valeur 0 garantie
- B. Une erreur de constructeur manquant
- C. Une violation NotBlank garantie
- D. Une référence circulaire

**Q17.** (1 réponse) Un champ JSON inconnu est reçu par le serializer du framework. Quel comportement correspond au défaut et au mode strict ?

- A. Toujours une erreur JSON de syntaxe
- B. Toujours ajouté comme propriété dynamique
- C. Ignoré par défaut ; `ExtraAttributesException` avec `allow_extra_attributes: false`
- D. Supprime le DTO entier sans exception

**Q18.** (1 réponse) `email` est une chaîne syntaxiquement JSON correcte mais invalide selon `Assert\Email`. Qui rejette sa signification métier ?

- A. JsonEncoder
- B. Uniquement ObjectNormalizer
- C. Le routeur
- D. Validator

**Q19.** (1 réponse) Une `PartialDenormalizationException` contient un objet partiel. Que faire avant toute action métier ?

- A. Refuser ce traitement et exploiter les chemins d’erreur
- B. Traiter l’objet car sa classe est correcte
- C. Ignorer toutes les erreurs nullables
- D. Le sérialiser pour remplir automatiquement les champs manquants

**Q20.** (1 réponse) Pourquoi un test de sortie doit-il vérifier l’absence d’un faux secret ?

- A. Les propriétés privées sont toujours exclues
- B. Les groupes filtrent même sans contexte
- C. Un getter ou un contexte oublié peut exposer une donnée sensible
- D. Le format JSON chiffre les chaînes

## Réponses

1. **B**. L'encoder fait tableau ↔ string.
2. **A, B, C**. (Et `YamlEncoder`.) Pas de protobuf natif.
3. **C**. `ObjectNormalizer` : -1000 (`PropertyNormalizer` -990).
4. **B**. Priorité 0 ⇒ avant les natifs (négatifs) ; l'autoconfiguration tagge les `NormalizerInterface`/`DenormalizerInterface`.
5. **B**.
6. **B**. Booléen = mise en cache du résultat de `supports*()`.
7. **B**. Sans `groups` dans le contexte, pas de filtrage.
8. **B**.
9. **B**. `CIRCULAR_REFERENCE_LIMIT => 1` dans `AbstractNormalizer`.
10. **A, B, C**. D est absurde (on perdrait la normalisation d'objets).
11. **B**.
12. **B**. Voir `serializer/CHANGELOG.md` 8.0.
13. **A, B, C**. `#[Groups]` reste (dans `Attribute\`).
14. **B**.
15. **A, B**. C et D concernent `#[MapRequestPayload]`, la direction inverse.


## Réponses raisonnées — 16 à 20

16. **B.** Le normalizer ne dispose pas de l’argument requis. A invente une valeur ; C relève du Validator et d’une contrainte explicite ; D exige un cycle d’objets.
17. **C.** Le contrôle porte sur les métadonnées d’attributs. A confond syntaxe et structure ; B n’est pas une promesse du normalizer ; D masque un échec qui doit être observable.
18. **D.** Le type chaîne peut être construit avant validation. A traite le format, B la structure et les types, C la sélection de route.
19. **A.** Un objet partiel n’est pas un contrat valide. B assimile type et validité ; C élimine de l’information arbitrairement ; D ne répare aucune entrée.
20. **C.** La forme publique doit être prouvée sur la sortie effective. A ignore les getters et normalizers ; B est faux sans contexte de groupes ; D confond encodage et chiffrement.
