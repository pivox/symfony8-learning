# Module 07 — Serializer avancé

[Parcours](../README.md) · [Exercices](exercices.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

## Objectifs, prérequis et repères

> Application : **Symfony 8.1.8, PHP ≥ 8.4**. La [certification Symfony](https://certification.symfony.com/exams/symfony.html) cible **8.0** : les mentions **8.1 uniquement** sont des compléments pour cette application, pas des acquis à attribuer à 8.0.

À la fin du module, tu dois séparer encodage, normalisation et validation, définir un contrat d'entrée strict, maîtriser l'imbrication et garantir qu'une sortie ne révèle aucun secret.

Prérequis : DTO typés, conteneur et résolution d'arguments HTTP. Le module [Validator](../12-validator/README.md) approfondit ensuite la partie contraintes métier ; il n'est pas nécessaire de l'avoir terminé pour commencer la normalisation. Serializer, PropertyAccess, PropertyInfo et Validator sont installés. Aucun modèle Doctrine n'est nécessaire ; privilégier des DTO `readonly` à constructeur promu.

Applique la [méthode de travail commune](../README.md) : prédire, exécuter, observer, expliquer. Les exemples ci-dessous illustrent un mécanisme ; les exercices restent à réaliser. Le profiler n'est pas installé : utiliser les commandes et les tests proposés. Son installation facultative passerait par `composer require --dev symfony/profiler-pack`.

Packages déjà installés : `symfony/serializer` (+ `property-access` et `property-info`, nécessaires à `ObjectNormalizer`). Pour le XML/YAML/CSV : `XmlEncoder` demande l'extension `ext-dom`, `YamlEncoder` `symfony/yaml` (installé), `CsvEncoder` rien de plus. Pour `#[MapRequestPayload]` : `symfony/validator` (installé).

## 1. Architecture : sérialisation = normalisation + encodage

```
objet  --normalize-->  tableau (scalaires)  --encode-->  string (json, xml, csv, yaml)
objet  <-denormalize-  tableau              <-decode---  string
```

- `Serializer` (`SerializerInterface::serialize()/deserialize()`, `NormalizerInterface`, `DenormalizerInterface`, `EncoderInterface`, `DecoderInterface`) orchestre **normalizers** (objet ↔ tableau) et **encoders** (tableau ↔ format).
- Injecte `SerializerInterface` (ou `NormalizerInterface`/`DenormalizerInterface` quand tu n'as besoin que d'une moitié).

```php
$json = $serializer->serialize($user, 'json', ['groups' => ['read']]);
$user = $serializer->deserialize($json, User::class, 'json');
$array = $serializer->normalize($user, null, ['groups' => ['read']]);
$users = $serializer->deserialize($json, User::class.'[]', 'json'); // collection (ArrayDenormalizer)
```

Encoders livrés : `JsonEncoder`, `XmlEncoder`, `CsvEncoder`, `YamlEncoder`. Chaque encoder déclare son format (`JsonEncoder::FORMAT = 'json'`).

## 2. Normalizers natifs et priorités

Enregistrés par FrameworkBundle (tags `serializer.normalizer`, valeurs vérifiées dans `serializer.php`) :

| Normalizer | Priorité |
|---|---|
| `UnwrappingDenormalizer` (`unwrap_path`) | 1000 |
| `ProblemNormalizer`, `UidNormalizer` | -890 |
| `DateTimeNormalizer` | -910 |
| `ConstraintViolationListNormalizer`, `MimeMessageNormalizer`, `DateTimeZoneNormalizer`, `DateIntervalNormalizer`, `FormErrorNormalizer`, `BackedEnumNormalizer`, `NumberNormalizer` | -915 |
| `DataUriNormalizer`, `TranslatableNormalizer` | -920 |
| `JsonSerializableNormalizer` | -950 |
| `PropertyNormalizer` | -990 |
| `ObjectNormalizer` | -1000 (le dernier recours) |

- Le **premier** normalizer (ordre de priorité décroissante) dont `supportsNormalization()` répond `true` est utilisé : un normalizer custom avec priorité par défaut (0) passe **avant** les natifs.
- `getSupportedTypes(?string $format): array` (obligatoire depuis 7.0) : retourne `['object' => true]`, `[Foo::class => true]`, `['*' => false]`... ; `true` = le résultat de `supports*()` peut être mis en cache (ne dépend que du type), `false` = dépend des données/contexte (exemple de la doc d'interface), `null` = type non supporté. Permet au Serializer d'éviter d'interroger tous les normalizers à chaque appel.
- `GetSetMethodNormalizer` existe mais n'est **pas** enregistré par défaut (absent de `serializer.php`).

### `ObjectNormalizer` vs `PropertyNormalizer` vs `GetSetMethodNormalizer`
- `ObjectNormalizer` : utilise `PropertyAccess` (getters, `isX`, `hasX`, `canX`, setters, propriétés publiques, ...) + `PropertyInfo` pour les types. C'est celui par défaut.
- `PropertyNormalizer` : lit/écrit **les propriétés** directement par réflexion, y compris `private`/`protected`, sans getters ; ignore propriétés `static`.
- `GetSetMethodNormalizer` : uniquement getters/setters (`getX`, `isX`, `hasX`, `setX`).
- Pour utiliser un autre que le défaut, déclare-le toi-même (avec priorité plus haute que -1000) ou crée un serializer nommé.

## 3. Contexte

Tableau d'options passé à chaque appel (`$serializer->serialize($o, 'json', $context)`), ou par défaut via `framework.serializer.default_context`, ou par attribut `#[Context]`. Constantes (vérifiées) :

| Clé | Rôle |
|---|---|
| `AbstractNormalizer::GROUPS` (`groups`) | filtrer par groupes |
| `ATTRIBUTES` (`attributes`) | liste blanche d'attributs (supporte l'imbriqué) |
| `IGNORED_ATTRIBUTES` | liste noire |
| `OBJECT_TO_POPULATE` | hydrater un objet existant |
| `ALLOW_EXTRA_ATTRIBUTES` | défaut `true` (les champs inconnus sont ignorés ; `false` → `ExtraAttributesException`) |
| `CIRCULAR_REFERENCE_LIMIT` | défaut **1** ; `CIRCULAR_REFERENCE_HANDLER` : callable |
| `DEFAULT_CONSTRUCTOR_ARGUMENTS`, `CALLBACKS`, `REQUIRE_ALL_PROPERTIES`, `FILTER_BOOL` | divers |
| `AbstractObjectNormalizer::ENABLE_MAX_DEPTH`, `MAX_DEPTH_HANDLER` | profondeur |
| `SKIP_NULL_VALUES`, `SKIP_UNINITIALIZED_VALUES`, `PRESERVE_EMPTY_OBJECTS`, `DEEP_OBJECT_TO_POPULATE`, `DISABLE_TYPE_ENFORCEMENT`, **`ENABLE_TYPE_CONVERSION`** (8.1) | |
| `DateTimeNormalizer::FORMAT_KEY` (`datetime_format`), `TIMEZONE_KEY`, `CAST_KEY`, `FORCE_TIMEZONE_KEY` | dates |
| `Serializer::EMPTY_ARRAY_AS_OBJECT` | `[]` → `{}` en JSON |
| `UnwrappingDenormalizer::UNWRAP_PATH` | désérialiser un sous-nœud |
| `XmlEncoder::ROOT_NODE_NAME`, `AS_COLLECTION`, `CsvEncoder::DELIMITER_KEY`, ... | encodage |

`ContextBuilder`s typés (`ObjectNormalizerContextBuilder`, `JsonEncoderContextBuilder`...) : alternative objet au tableau.

## 4. Attributs de mapping

```php
use Symfony\Component\Serializer\Attribute\{Groups, SerializedName, SerializedPath, Ignore, MaxDepth, Context, DiscriminatorMap};

class Author
{
    #[Groups(['read', 'write'])] public string $name;
    #[Groups(['read'])] #[SerializedName('mail')] public string $email;           // nom exposé
    #[Ignore] public string $secret;                                               // jamais (dé)sérialisé
    #[Groups(['read'])] #[SerializedPath('[profile][age]')] public int $age;       // chemin imbriqué (PropertyAccess)
    #[Groups(['read'])] #[MaxDepth(1)] public ?Author $parent = null;
    #[Context([DateTimeNormalizer::FORMAT_KEY => 'Y-m-d'])] public \DateTimeImmutable $born;
}

#[DiscriminatorMap(typeProperty: 'type', mapping: ['car' => Car::class, 'bike' => Bike::class])]
abstract class Vehicle {}
```

(Namespace `Symfony\Component\Serializer\Attribute`, les alias `Annotation\` ont été dépréciés en 7.4 et supprimés en 8.0.)

- `#[Groups]` : à utiliser **avec** le contexte `groups` ; sans `groups` dans le contexte **tout** est (dé)sérialisé, y compris les propriétés groupées.
- `#[Context]` : `context`, `normalizationContext`, `denormalizationContext`, `groups` (le contexte ne s'applique qu'à ces groupes).
- `#[MaxDepth]` n'a d'effet qu'avec `enable_max_depth: true` dans le contexte.
- `#[DiscriminatorMap]` : polymorphisme ; `defaultType` en option.
- `#[ExtendsSerializationFor]` (7.4) : déclarer les attributs d'une classe tierce depuis une autre classe.
- Mappings XML/YAML possibles (`framework.serializer.mapping.paths`).
- En 8.1 sans FullStack, `framework.serializer.enable_attributes` vaut `true` par défaut.

## 5. Références circulaires, profondeur

```php
$serializer->normalize($a, null, [
    AbstractNormalizer::CIRCULAR_REFERENCE_HANDLER => fn (object $o) => $o->getId(),
]);
```

- Sans handler : `CircularReferenceException` dès que la même instance revient plus de `circular_reference_limit` (**1**) fois.
- Alternatives : `#[MaxDepth]` + `enable_max_depth`, groupes qui coupent la boucle, `ignored_attributes`.
- Configuration globale : `framework.serializer.circular_reference_handler` et `max_depth_handler` (services).

## 6. Name converters

`NameConverterInterface` : `normalize()`/`denormalize()` — signature **modifiée en 8.0** : `(string $propertyName, ?string $class = null, ?string $format = null, array $context = []): string`; `AdvancedNameConverterInterface` supprimé.
Natifs : `CamelCaseToSnakeCaseNameConverter`, `SnakeCaseToCamelCaseNameConverter`, `MetadataAwareNameConverter` (applique `#[SerializedName]`, enveloppe un autre converter). Config : `framework.serializer.name_converter: serializer.name_converter.camel_case_to_snake_case`.

## 7. Normalizer / Encoder custom

```php
final class MoneyNormalizer implements NormalizerInterface, DenormalizerInterface
{
    public function normalize(mixed $data, ?string $format = null, array $context = []): string { return $data->format(); }
    public function supportsNormalization(mixed $data, ?string $format = null, array $context = []): bool { return $data instanceof Money; }
    public function denormalize(mixed $data, string $type, ?string $format = null, array $context = []): Money { ... }
    public function supportsDenormalization(mixed $data, string $type, ?string $format = null, array $context = []): bool { return Money::class === $type; }
    public function getSupportedTypes(?string $format): array { return [Money::class => true]; }
}
```

- Autoconfiguré (tag `serializer.normalizer`) car il implémente `NormalizerInterface`/`DenormalizerInterface` (vérifié : `registerForAutoconfiguration`). Idem `EncoderInterface`/`DecoderInterface` → `serializer.encoder`.
- Déléguer aux autres normalizers : `NormalizerAwareInterface` + `NormalizerAwareTrait` (idem pour `DenormalizerAware...`), puis `$this->normalizer->normalize($sub, $format, $context)` ; pense à éviter la récursion infinie (marqueur dans le contexte).
- Décorer le normalizer natif (module 03) : `#[AsDecorator('serializer.normalizer.object')]`.
- `CustomNormalizer` + `NormalizableInterface` / `DenormalizableInterface` : l'objet se normalise lui-même.
- Serializers **nommés** : `framework.serializer.named_serializers` (name_converter, `include_built_in_normalizers/encoders`, `default_context` par serializer) ; injection par `#[Target('nom.serializer')]`.

## 8. Dans les contrôleurs

- `#[MapRequestPayload]` / `#[MapQueryString]` : désérialisent + valident le corps / la query (module 04). Options `serializationContext`, `validationGroups`, `acceptFormat`, `resolver`.
- `#[Serialize]` (HttpKernel **8.1**) : le contrôleur retourne un objet/tableau, l'attribut le sérialise en réponse : `new Serialize(code: 200, headers: [], context: [])` ; écouté sur `kernel.view`.
- Erreurs de désérialisation : `NotNormalizableValueException` (type incorrect), `MissingConstructorArgumentsException`, `ExtraAttributesException`, `PartialDenormalizationException` (avec `COLLECT_DENORMALIZATION_ERRORS`).

## Ce qui change vs Symfony 6/7
- **8.0** (`serializer/CHANGELOG.md`) : suppression de `CsvEncoder::ESCAPE_CHAR_KEY` ; de `AbstractNormalizerContextBuilder::withDefaultContructorArguments()` (faute de frappe) → `withDefaultConstructorArguments()` ; signature de `NameConverterInterface::normalize/denormalize` ; `AdvancedNameConverterInterface` supprimé ; **namespace `Annotation\` supprimé** (utiliser `Attribute\`) ; les getters des classes d'attributs supprimés au profit de propriétés publiques ; `ClassMetadataFactoryCompiler`/`CompiledClassMetadataFactory` supprimés.
- **8.1** : `AbstractObjectNormalizer::ENABLE_TYPE_CONVERSION` ; option `COLLECT_EXTRA_ATTRIBUTES_ERRORS` ; messages d'erreur d'enum améliorés ; dépréciation du parsing de date avec format par défaut raté ; `PartialDenormalizationException::getErrors()` déprécié → `getNotNormalizableValueErrors()` ; `#[Serialize]` côté HttpKernel.
- 7.x : `getSupportedTypes()` devenu obligatoire (7.0), `#[ExtendsSerializationFor]` (7.4), `can*()` dans `AttributeLoader` (7.4).

## Piège certification
- Normalisation ≠ encodage : un normalizer peut produire un tableau ou un scalaire (dont une string) : cela ne constitue pas nécessairement un document encodé, l'encoder reçoit des scalaires/tableaux.
- Sans `groups` dans le contexte, `#[Groups]` ne filtre rien.
- `#[MaxDepth]` inactif sans `enable_max_depth`.
- `circular_reference_limit` = 1 par défaut ; une 2e occurrence de la même instance → exception.
- `ObjectNormalizer` passe par les accesseurs (getter/setter/hasser/isser), `PropertyNormalizer` par les propriétés.
- Priorité : plus grande = interrogé plus tôt ; `ObjectNormalizer` à -1000 est le dernier.
- `getSupportedTypes` renvoie des **booléens de cache**, pas des priorités.
- `allow_extra_attributes` vaut `true` par défaut en dénormalisation.
- `deserialize()` d'une collection : type `Foo[]`.
- `SerializedName` ne renomme que dans le sens (dé)normalisation ; il s'applique via `MetadataAwareNameConverter`.

## Observer dans l'app
- `bin/console debug:serializer 'App\Serializer\Author'` : propriétés, groupes, noms sérialisés (commande `debug:serializer`, vérifiée).
- `bin/console debug:container --tag=serializer.normalizer` / `--tag=serializer.encoder` ; `debug:container serializer.normalizer.object`.
- `bin/console debug:config framework serializer`.
- `bin/console debug:autowiring Serializer` (injection par type, serializers nommés).
- Après installation facultative du profiler : profiler → onglet **Serializer** (si `framework.serializer` en debug : normalizers utilisés, temps, données in/out ; option `collect_serializer_data`).
- Tests : `KernelTestCase` + `self::getContainer()->get('serializer')` ; `assertJsonStringEqualsJsonString`.

## Doc officielle
- https://symfony.com/doc/current/serializer.html
- https://symfony.com/doc/current/serializer/custom_normalizer.html
- https://symfony.com/doc/current/serializer/custom_context_builders.html
- https://symfony.com/doc/current/serializer/custom_name_converter.html
- https://symfony.com/doc/current/serializer/encoders.html
- https://symfony.com/doc/current/serializer/normalizers.html
- https://symfony.com/doc/current/reference/configuration/framework.html#serializer

## 9. Définir un contrat d'entrée avant le JSON

Quatre questions indépendantes doivent être tranchées : la donnée est-elle syntaxiquement décodable, sa structure correspond-elle au DTO, les types sont-ils constructibles, les valeurs sont-elles acceptables métier ? Le Serializer couvre les trois premières ; Validator couvre la dernière. `"email":"pas-un-email"` reste une chaîne parfaitement dénormalisable, même si `#[Assert\Email]` la refuse ensuite.

```php
use Symfony\Component\Validator\Constraints as Assert;

final readonly class DeliveryInput
{
    public function __construct(
        #[Assert\NotBlank] public string $recipient,
        #[Assert\Positive] public int $quantity,
        public ?string $note = null,
    ) {}
}
```

Dans un service utilisant le serializer configuré de l'application, dénormaliser `['recipient' => 'Ada', 'quantity' => 2]` construit un objet dont `note` vaut `null`. `quantity` absent sans défaut provoque une `MissingConstructorArgumentsException`. Une valeur `[]` pour `quantity` ne représente pas un entier : attendre une erreur de dénormalisation, pas une violation `Positive`. La chaîne numérique `"2"` est un mauvais témoin universel : les conversions dépendent du format et du contexte ; choisir un tableau pour un test de type non ambigu.

Un champ inconnu est ignoré par défaut. Avec `AbstractNormalizer::ALLOW_EXTRA_ATTRIBUTES => false`, le normalizer peut lever `ExtraAttributesException` ; la détection exige une factory de métadonnées (présente dans le serializer du framework, à fournir si tu assembles les composants toi-même). `REQUIRE_ALL_PROPERTIES` concerne notamment les paramètres de constructeur nullables sans défaut, que le normalizer peut autrement remplir avec `null` ; ce n'est pas un remplacement de `NotBlank`.

### Imbrication et erreurs partielles

Une propriété typée `AddressInput` donne une cible concrète au normalizer pour un sous-objet. Un simple `array` ne précise pas le type des éléments : `list<AddressInput>` nécessite des métadonnées que l'extracteur PropertyInfo sait lire. Vérifier les extracteurs réellement installés avant de promettre une inférence depuis PHPDoc. Préférer d'abord un sous-objet directement typé puis expérimenter les collections.

```php
use Symfony\Component\Serializer\Normalizer\DenormalizerInterface;

$context = [DenormalizerInterface::COLLECT_DENORMALIZATION_ERRORS => true];
// Passer ce contexte à deserialize() pour collecter plusieurs erreurs de type.
```

Une `PartialDenormalizationException` expose des erreurs et éventuellement des données partielles : celles-ci servent au diagnostic, **pas** au traitement métier. En **8.0**, lire `getErrors()` ; en **8.1 uniquement**, utiliser `getNotNormalizableValueErrors()` (`getErrors()` est dépréciée). Relever le `getPath()` de chaque erreur permet de rattacher un message à `address.city`. Le nombre d'erreurs collectées ne couvre pas automatiquement toutes les catégories d'erreurs : les champs supplémentaires ont une option de collecte distincte en **8.1 uniquement**.

Dans un contrôleur, préférer `#[MapRequestPayload]` avec `serializationContext` et des DTO validés ; ne pas appeler `json_decode()` à la main. Tester séparément JSON malformé, format non accepté, type invalide et contrainte métier. Le resolver adapte certaines erreurs en violations et en statut HTTP : on ne peut pas déduire le statut du seul nom d'exception du Serializer. Voir le [module diagnostic et tests](../14-tests-diagnostic/README.md).

### Une sortie est aussi un contrat

Un champ privé avec getter peut être normalisé. Un mot de passe hashé reste une information sensible. Les groupes ne filtrent que si l'appel fournit un contexte `groups` ; `#[Ignore]` agit avec les métadonnées, mais retire aussi le champ de la dénormalisation. Un DTO de réponse dédié sans secret rend l'intention plus robuste. Tester **les clés absentes**, et pas seulement la présence de `id` ou `name`. Comparer une normalisation avec `groups: ['public']` et sans groupes sur une fixture contenant un faux secret.

Observation : `debug:serializer` montre les métadonnées de la classe, pas une preuve de la réponse effectivement exposée. Compléter avec un appel au service dans `KernelTestCase` et un test HTTP qui contrôle le JSON final. Sources locales : `AbstractNormalizer`, `AbstractObjectNormalizer`, `RequestPayloadValueResolver` ; [Serializer 8.0](https://symfony.com/doc/8.0/serializer.html).
