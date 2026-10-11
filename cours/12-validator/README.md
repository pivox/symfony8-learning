# Module 12 — Validator : règles, contextes et chemins d'erreur

[Parcours](../README.md) · [Exercices](exercices.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Concevoir un contrat de validation réutilisable en HTTP, en commande et dans un service.
Savoir choisir groupes, séquences et cascade ; produire des erreurs dont le chemin désigne le champ concerné.
Prérequis : DTO PHP et [DI](../03-dependency-injection/README.md). Pour l'exercice HTTP, les bases de `MapRequestPayload` du [module 04](../04-http-rfc9110/README.md) suffisent ; [Serializer](../07-serializer/README.md) est un approfondissement complémentaire, pas un préalable à toute validation.
Fil rouge : inscription publique en mémoire, contextes brouillon/publication, sans compte enregistré ni mécanisme d'authentification.
Code futur : `src/Validation/`, tests : `tests/Validation/` ; aucun stockage à installer.
Validator et Serializer sont déjà présents. Symfony **8.1.8** est exécuté ; le socle de certification reste **8.0**.

## 1. Contrainte, validateur et résultat

Une contrainte décrit une règle et ses options. Un validateur exécute cette règle sur une valeur et dans un contexte.
`ValidatorInterface::validate($value, $constraints, $groups)` retourne une liste de violations ; ce n'est pas un booléen.
Un objet invalide ordinaire ne déclenche pas une exception : parcourir ou compter la liste fait partie du contrat.
Une contrainte mal configurée ou un type incompatible avec son validateur peut, lui, provoquer une exception.
Le Validator ne transforme pas une adresse en minuscules et ne persiste rien : validation, normalisation et écriture sont distinctes.

```php
use Symfony\Component\Validator\Constraints as Assert;

final readonly class ContactInput
{
    public function __construct(
        #[Assert\NotBlank]
        #[Assert\Email]
        public string $email,
        #[Assert\Length(min: 2, max: 60)]
        public string $displayName = '',
    ) {}
}
```

Ici `Length(min: 2)` rejette `''`, mais accepte `null` sur un type nullable ; `Email` accepte aussi une chaîne vide.
Ajouter `NotBlank` pour interdire le vide ; `NotNull` interdit seulement `null`, pas `''`.
`Email` vérifie une forme, pas l'existence d'une boîte ni la capacité à recevoir un message.
Initialiser les propriétés évite de confondre une donnée absente avec une propriété typée non initialisée, traitée comme `null` par validation.

## 2. Observer les violations

```php
use Symfony\Component\Validator\Validator\ValidatorInterface;

// $validator est injecté dans le service appelant.
$violations = $validator->validate(new ContactInput('incorrect'));
foreach ($violations as $violation) {
    $path = $violation->getPropertyPath();
    $message = $violation->getMessage();
    $code = $violation->getCode();
}
```

Le message est destiné à l'humain et peut être traduit ; le code identifie la catégorie de violation.
Un test robuste vérifie chemin, code et nombre attendu, sans dépendre uniquement d'un texte traduit.
`getInvalidValue()` peut contenir une donnée sensible : ne pas tout recopier dans les logs ni les réponses.
Une erreur au niveau classe a un chemin vide tant que le validateur ne la rattache pas à une propriété.
Deux contraintes peuvent échouer sur la même propriété : une liste conserve cette multiplicité.

## 3. Groupes : sélectionner, pas ordonner

Sans groupe explicite, `validate()` utilise `Default`. Une contrainte sans groupe appartient à ce groupe.
Une contrainte assignée uniquement à `publish` n'est pas exécutée par une validation `Default` ordinaire.

```php
#[Assert\NotBlank(groups: ['publish'])]
#[Assert\Length(max: 60)]
public string $displayName;

// Dans le service, avec un DTO effectivement construit :
$violations = $validator->validate($input, null, ['Default', 'publish']);
```

Passer `['Default', 'publish']` sélectionne les deux ensembles, sans condition « seulement si Default réussit ».
Choisir les groupes dans la couche appelante exprime le contexte : brouillon, publication, import.
Ce ne sont ni des groupes Serializer ni des rôles Security, même si leurs chaînes peuvent se ressembler.
Éviter une multiplication de groupes qui reproduit toutes les combinaisons d'un processus métier.

## 4. Séquences : avancer seulement après succès

```php
use Symfony\Component\Validator\Constraints\GroupSequence;

$violations = $validator->validate($input, null,
    new GroupSequence(['Default', 'publish']));
```

Si `Default` échoue, `publish` n'est pas exécuté. Toutes les contraintes du groupe courant restent susceptibles d'être évaluées.
Une séquence n'arrête donc pas nécessairement à la première violation individuelle.
Sur une classe `SignupInput`, `#[Assert\GroupSequence(['SignupInput', 'Strict'])]` remplace sa validation `Default`.
Le nom court de la classe représente alors ses contraintes ordinaires ; mettre `Default` dans cette séquence créerait une récursion interdite.
La séquence de classe doit contenir le groupe portant le nom de cette classe.
`GroupSequenceProviderInterface` permet une séquence dynamique, avec `#[Assert\GroupSequenceProvider]`.
`Sequentially` enchaîne des contraintes sur une même valeur ; ce n'est pas un synonyme de groupes de classe.

## 5. Objets imbriqués et collections

Une propriété objet typée n'active pas automatiquement la validation de son contenu.
Placer `#[Assert\Valid]` sur `address` déclenche la cascade vers les contraintes du DTO adresse.
Pour une collection de DTO sous `contacts`, les chemins deviennent par exemple `contacts[0].email`.
`Valid` n'interdit pas `null` : ajouter `NotNull` si l'objet imbriqué est obligatoire.
`Count` contraint la taille d'une collection ; `All` applique des contraintes à chaque valeur.
`Collection` décrit les clés d'un tableau et utilise `Required`/`Optional` pour leurs présences.
Ne pas confondre `Optional` dans `Collection` avec « une chaîne vide est valide » : la présence et le contenu sont différents.
La cascade doit aussi être testée avec les groupes réellement transmis ; une règle enfant dans `publish` n'est pas une règle `Default`.

```php
#[Assert\Valid]
#[Assert\NotNull]
public ?AddressInput $address;

#[Assert\All([new Assert\Email()])]
public array $notificationEmails;
```

Ces fragments illustrent des propriétés de DTO ; ajouter constructeur promu et initialisation dans une vraie classe.

## 6. Règle personnalisée et erreurs ciblées

Pour une règle réutilisable, écrire une classe attribut qui étend `Constraint` et une classe qui étend `ConstraintValidator`.
La convention `AllowedDomain` → `AllowedDomainValidator` suffit si `validatedBy()` n'est pas redéfini.
La contrainte déclare ses options et messages ; le validateur reçoit par DI les services nécessaires.
Ne pas mettre de service dans la contrainte : les métadonnées doivent pouvoir être mises en cache.
Dans `validate(mixed $value, Constraint $constraint): void`, vérifier la classe de contrainte et gérer les types acceptés.
Laisser `null`/vide aux contraintes de présence si c'est le contrat choisi ; le tester explicitement.

```php
// Fragment interne d'un validateur de classe :
$this->context->buildViolation('Les valeurs ne concordent pas.')
    ->atPath('confirmation')
    ->addViolation();
```

`atPath()` est relatif au contexte courant ; sur une règle de propriété, ajouter son propre nom une seconde fois produit un mauvais chemin.
`Callback` convient à une règle locale croisant des propriétés ; une contrainte de classe partagée convient à une règle réutilisée.
L'autoconfiguration enregistre les validateurs services ; observer le tag `validator.constraint_validator` avant de configurer manuellement.

## 7. Frontière HTTP et diagnostic

`#[MapRequestPayload(validationGroups: ['Default', 'publish'])]` applique le contrat au DTO d'une action.
Le mapping s'occupe de désérialiser puis valider ; ne pas reparcourir manuellement le JSON dans le contrôleur.
Une donnée impossible à désérialiser et une donnée bien typée mais invalide ne sont pas le même échec ; couvrir les deux.
Le statut par défaut d'une validation de payload échouée est **422** ; déclarer le format JSON de la route pour le contrat JSON.

```bash
symfony console debug:validator 'App\Validation\SignupInput'
symfony console debug:container --tag=validator.constraint_validator
symfony console debug:autowiring ValidatorInterface
symfony console debug:config framework validation
php bin/phpunit tests/Validation
```

Lire `var/log/dev.log` en cas d'exception. Le profiler facultatif exige d'abord `composer require --dev symfony/profiler-pack`.
Tests de service : `KernelTestCase` avec `ValidatorInterface` du conteneur ; tests HTTP : `WebTestCase`.
Ces tests observent la configuration réelle, y compris attributs, groupes et injection du validateur personnalisé.

## 8. Versions et sources officielles

**8.0 :** les anciennes options sous forme de tableau de configuration des contraintes ne sont plus acceptées ; préférer les arguments nommés.
**8.1 hors certification :** `findByCodes()`, contrainte `Xml`, évolution vers `validateInContext()` dans l'interface des validateurs.
Étendre `ConstraintValidator` reste la voie enseignée : sa classe de base adapte le contexte et laisse implémenter `validate()`.
Ne pas copier un appel direct à `initialize()/validate()` sur l'interface depuis un ancien exemple ; appeler le service `ValidatorInterface`.
Consulter [Validation 8.0](https://symfony.com/doc/8.0/validation.html), [groupes](https://symfony.com/doc/8.0/validation/groups.html), [séquences](https://symfony.com/doc/8.0/validation/sequence_provider.html) et [contraintes personnalisées](https://symfony.com/doc/8.0/validation/custom_constraint.html).
Vérification locale : `validator/CHANGELOG.md`, `Constraints/GroupSequence.php`, `ConstraintValidator.php` sous `vendor/symfony/`.
Suite : réutiliser ces règles dans une [commande](../13-console/README.md), puis travailler les [tests](../14-tests-diagnostic/README.md).
