# Exercices — Module 07

[Parcours](../README.md) · [Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

Code dans `src/Serializer/` (namespace `App\Serializer`), tests dans `tests/Serializer/` (namespace `App\Tests\Serializer`) : `php bin/phpunit tests/Serializer`. Écris le test d'abord. Pour un serializer **sans** kernel, assemble-le à la main (`new Serializer([normalizers], [encoders])`) ; pour tester la config d'app, `KernelTestCase` + `getContainer()->get('serializer')`.

## 1. Aller-retour JSON
Crée un DTO `Author` (propriétés typées, constructeur promu) et vérifie `serialize` → `deserialize`.
- Tests : `Author` identique après aller-retour ; `deserialize` d'un tableau `Author[]` ; champ inconnu dans le JSON ⇒ ignoré (puis `allow_extra_attributes: false` ⇒ `ExtraAttributesException`).
- Réflexion : quel normalizer et quel encoder interviennent ? Pourquoi `ObjectNormalizer` a-t-il besoin de `property-access` ?

## 2. Groupes
Ajoute `#[Groups(['read'])]` / `#[Groups(['write'])]` à `Author` (id lecture seule, `password` écriture seule).
- Tests : `groups: ['read']` ⇒ pas de `password` ; `groups: ['write']` ⇒ l'inverse ; **sans** `groups` ⇒ ?
- Réflexion : pourquoi ne jamais exposer l'entité brute ?

## 3. Noms et chemins
`#[SerializedName]` (ex. `email` ↔ `mail`), `#[SerializedPath]` pour imbriquer `[profile][age]`, `#[Ignore]` sur un champ interne, puis `CamelCaseToSnakeCaseNameConverter` en parallèle.
- Tests : JSON attendu exact ; round-trip OK.
- Réflexion : l'ordre d'application `SerializedName` vs name converter global ? (utilise `MetadataAwareNameConverter` et observe). Test à écrire pour le prouver.

## 4. Dates, enums et contexte par attribut
Ajoute un `DateTimeImmutable` et un enum backed (`Status`). Formate la date avec `#[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])]`.
- Tests : format de sortie ; désérialisation d'une valeur d'enum invalide ⇒ `NotNormalizableValueException` (lis le message).
- Réflexion : normalizer concerné pour chaque type ; comment forcer un fuseau horaire en dénormalisation ?

## 5. Références circulaires et profondeur
Modèle `Category` (parent/enfants) avec référence bidirectionnelle.
- Tests : sans handler ⇒ `CircularReferenceException` ; avec `CIRCULAR_REFERENCE_HANDLER` ⇒ identifiant ; avec `#[MaxDepth(1)]` + `ENABLE_MAX_DEPTH` ⇒ profondeur coupée ; **sans** `ENABLE_MAX_DEPTH` ⇒ `MaxDepth` sans effet (le prouver).
- Réflexion : limite par défaut ? Que change `CIRCULAR_REFERENCE_LIMIT => 2` ?

## 6. Normalizer custom
Écris `MoneyNormalizer` (`Money` value object ⇄ `"12.50 EUR"`), avec `getSupportedTypes()` correct.
- Tests : normalisation, dénormalisation, mauvaise chaîne ⇒ `NotNormalizableValueException`, `supports*` pour d'autres types.
- Dans l'app : vérifie qu'il est auto-taggé (`debug:container --tag=serializer.normalizer`) et regarde sa **priorité** par rapport à `ObjectNormalizer`.
- Réflexion : que signifie `true`/`false`/`null` dans `getSupportedTypes()` ? Quel est le risque d'un `['*' => true]` ?

## 7. Normalizer qui délègue + décoration
Écris un `ApiEnvelopeNormalizer` (NormalizerAwareInterface/Trait) qui enveloppe la normalisation d'un `Author` : `{ "data": {...}, "meta": {...} }` en déléguant le contenu aux autres normalizers.
- Tests : structure d'enveloppe ; aucune récursion infinie (pose un marqueur dans le contexte).
- Réflexion : décorer (`#[AsDecorator]`) vs ajouter un normalizer prioritaire : avantages/inconvénients ?

## 8. Encoder et polymorphisme
Implémente un petit `EncoderInterface`/`DecoderInterface` pour un format maison (ex. `key=value` par ligne) et teste `serialize($obj, 'kv')`. Puis ajoute `#[DiscriminatorMap]` sur une hiérarchie `Vehicle`.
- Tests : round-trip `kv` ; désérialisation du bon sous-type d'après le champ discriminant ; type inconnu ⇒ exception.
- Observation : `debug:serializer 'App\Serializer\...\Vehicle'`.
- Réflexion : dans un contrôleur, comment `#[MapRequestPayload]` et `#[Serialize]` (8.1) réutilisent-ils ce serializer ? Où passer `groups` ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.

## Critères complémentaires pour les exercices 1–8

Appliquer la [méthode commune](../README.md) et distinguer test du Serializer, du Validator et test HTTP.

| Exercice | Acceptation supplémentaire |
|---|---|
| 1 | Comparer valeurs et types après aller-retour, pas l'identité d'instance ; fixture de collection vide puis deux auteurs. |
| 2 | Sur une fixture au secret connu, vérifier son absence en sortie `read`, puis démontrer sa présence sans groupes ; ne pas exposer ce scénario volontairement fragile comme endpoint public. |
| 3 | Comparer les clés exactes (`mail`, `profile.age`) ; un champ `Ignore` ne peut pas participer à un aller-retour identique : lui donner un défaut ou l'exclure de cette égalité. |
| 4 | Fixer instant et fuseau ; enum valide reconstruit en instance d'enum, enum inconnu refusé. |
| 5 | Tester une véritable réutilisation de la même instance ; une structure profonde sans cycle ne prouve pas une référence circulaire. |
| 6 | `12.50 EUR` reconstruit le montant exact selon le contrat du value object ; devise ou syntaxe invalide refusée, type non ciblé laissé aux autres normalizers. |
| 7 | Enveloppe avec exactement `data` et `meta`, aucun second emballage ; si `supportsNormalization()` dépend du marqueur de contexte, résultat non cacheable (`false` dans `getSupportedTypes`). |
| 8 | Définir l'échappement des `=` et retours à la ligne du format `kv`, ou les refuser explicitement ; valeur discriminante inconnue → exception vérifiée. |

## 9. Parcours guidé — matrice des entrées

Créer un DTO `readonly` à constructeur promu avec `name: string`, `count: int`, `note: ?string = null`. Utiliser le Serializer du conteneur dans `KernelTestCase`.

- Acceptation : `{"name":"Ada","count":2}` → objet et note `null` ; absence de `count` → `MissingConstructorArgumentsException` ; `count: []` → `NotNormalizableValueException` ; `unexpected: true` → ignoré par défaut puis `ExtraAttributesException` en contexte strict.
- Ajouter `#[Assert\Positive]` : `count: -1` doit être dénormalisé, puis produire une violation à la validation. Expliquer les deux étapes.
- Indice : tester chaque catégorie avec une fixture qui n'a qu'un seul défaut pour ne pas masquer les autres.

## 10. Parcours intermédiaire — sous-objets et erreurs agrégées

Ajouter un sous-objet typé `AddressInput` (city et postalCode chaînes), puis un champ entier `count`. Fournir des tableaux à la place de `address.city` et `count` et activer la collecte.

- Acceptation : entrée correcte → véritable `AddressInput` ; entrée incorrecte → `PartialDenormalizationException` avec deux chemins d'erreur pertinents ; aucun objet partiel transmis au service métier.
- Lire les erreurs avec l'API correspondant à la version : `getErrors()` en 8.0, `getNotNormalizableValueErrors()` en 8.1.
- Indice : conserver `postalCode` sous forme de chaîne pour les zéros initiaux ; distinguer classe du sous-objet et tableau de valeurs.

## 11. Parcours avancé — contrat HTTP et données sensibles

Exposer une route d'inscription avec `#[MapRequestPayload]` et répondre avec un DTO public dédié contenant seulement `id` et `displayName`. Pour le contrat de cet exercice, définir explicitement les erreurs de validation à 422 et la sortie réussie à 201.

- Acceptation : données valides → 201 et exactement ces deux clés ; JSON malformé → 400 ; type de contenu non accepté → 415 ; donnée métier invalide → 422 ; pas d'appel au service métier sur ces échecs.
- Mettre dans la fixture un faux mot de passe, un hash et un token : aucun ne doit apparaître dans la réponse, y compris imbriqué.
- Indice : groupes de lecture et DTO de sortie sont deux protections différentes ; tester l'objet réellement passé au serializer. `#[Serialize]` serait un complément **8.1 uniquement**, pas requis pour cet exercice 8.0.

## Défi final — import de destinataires

Concevoir un contrat d'import avec destinataire et adresse imbriquée, rejeter les champs inattendus, puis produire une réponse publique. Choisir et documenter le comportement lorsqu'un élément de collection échoue ; aucun traitement partiel implicite.

Livrables : tableau entrée → étape en échec → exception/statut, tests de service et HTTP, exemple de réponse réussie et refusée. Acceptation : types et imbrication conservés, valeurs métier validées, chemins d'erreurs exploitables, zéro secret exposé, zéro appel métier lorsque l'import est refusé. Indice final : vérifier séparément décodage, dénormalisation et validation avant d'assembler la route.

## 12. Forme JSON d'un profil incomplet — intermédiaire

Durée indicative : 35–45 min. Prérequis : exercices 1–2 et [états absents](approfondissement.md).
Scénario : un client distingue une note inconnue, une liste vide et un objet de préférences vide.
Créer une fixture `ProfileOutput` avec `note = null`, `tags = []`, un objet `preferences` sans attribut exposé et une propriété typée `alias` non initialisée, uniquement pour l'expérience.
Appeler réellement `serialize()` du conteneur dans `KernelTestCase` ; produire trois sorties avec contextes différents.

- Réussite : contexte ordinaire → `note` présente à `null`, `tags` vaut `[]`, `alias` absent ; contexte omettant les nulls → `note` absente.
- Réussite : avec préservation des objets vides, `preferences` est `{}` et `tags` reste `[]` ; comparer les types JSON, pas seulement deux tableaux PHP associatifs.
- Cas négatif : désactiver l'omission des propriétés non initialisées et constater la classe d'exception effective ; la propriété ne doit pas devenir silencieusement une chaîne vide.
- Preuve : tableau contexte → clés présentes → forme de `preferences` et `tags`, plus assertions sur le service réellement appelé.
- Indice 1 : absence, `null` et tableau vide sont trois états différents ; `isset()` ne distingue pas les deux premiers.
- Indice 2 : examiner `SKIP_UNINITIALIZED_VALUES`, `SKIP_NULL_VALUES` et `PRESERVE_EMPTY_OBJECTS` dans `AbstractObjectNormalizer`.

## 13. Modifier une fiche sans perdre son adresse — avancé

Durée indicative : 45–60 min. Prérequis : sous-objets typés, exercice 10 et [mise à jour d'objet](approfondissement.md).
Scénario : une fiche mutable contient `name = 'Ada'`, `note = 'porte B'` et une adresse mutable `city = 'Lyon'`, `postalCode = '69001'`.
Ces deux objets sont des modèles d'expérience modifiables ; conserver les DTO d'entrée `readonly` dans le reste du parcours.
Appliquer via le Serializer une première entrée `{"name":"Lin"}`, puis, sur une fixture neuve, `{"address":{"city":"Nantes"}}`.
Donner aux champs de l'adresse des défauts explicites pour pouvoir comparer remplacement et réutilisation sans erreur de constructeur.

- Réussite : mise à jour racine → instance racine identique, nom `Lin`, note inchangée et adresse inchangée.
- Réussite : sans mise à jour profonde, le sous-objet reçu remplace l'adresse ; avec elle, même instance d'adresse, ville `Nantes` et code postal `69001` conservé.
- Cas négatif : `{"name":"Lin","address":{"city":[]}}` échoue sur le type ; relever l'état de la racine après l'échec et démontrer qu'aucun traitement métier ultérieur n'est appelé.
- Livraison : expliquer pourquoi la dénormalisation dans un objet existant ne garantit pas une opération atomique, puis proposer en prose la frontière d'un DTO d'entrée distinct.
- Indice 1 : conserver les références de la racine et de l'adresse avant l'appel ; comparer `===` en plus des valeurs.
- Indice 2 : combiner `OBJECT_TO_POPULATE` et, seulement pour la variante concernée, `DEEP_OBJECT_TO_POPULATE` ; ne pas extrapoler ce dernier aux collections.
