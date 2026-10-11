# Approfondissement — structures et composition de contraintes

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Le cours valide des DTO et leurs graphes ; cette fiche décrit les tableaux à clés attendues et la composition locale de règles.
Socle **Symfony 8.0**, vérifié dans **8.1.8** ; les nouvelles méthodes 8.1 ne sont pas nécessaires.
Prérequis : `ValidatorInterface`, violations, groupes et chemins d'erreur.
Les données viennent de fixtures déterministes ; aucun accès réseau ni stockage n'intervient.

## 1. Collection décrit une structure, pas uniquement une liste

`Collection` associe à chaque clé d'un tableau les contraintes qui doivent s'y appliquer.
Le nom peut tromper : il ne signifie pas « répéter cette règle sur tous les objets ».
`All` répond à ce dernier besoin ; `Valid` déclenche la validation de DTO imbriqués.
Une règle de structure est utile pour un tableau de paramètres déjà obtenu par l'appelant.
Elle ne décode pas JSON et ne remplace pas `MapRequestPayload` à la frontière HTTP.

```php
use Symfony\Component\Validator\Constraints as Assert;

$shape = new Assert\Collection(fields: [
    'city' => new Assert\Required([new Assert\NotBlank()]),
    'comment' => new Assert\Optional([new Assert\Length(max: 40)]),
]);
// À passer au Validator pour examiner une valeur tableau.
```

Avec les options par défaut, une clé obligatoire absente et une clé inconnue produisent des violations.
Un champ déclaré sans enveloppe `Required` ou `Optional` est traité comme obligatoire.
`Optional` permet l'absence de la clé ; lorsqu'elle existe, ses contraintes s'exécutent normalement.
Ainsi une clé facultative avec `NotBlank` reste refusée si elle est fournie vide.
`Required` impose la présence de la clé, pas automatiquement une valeur non vide.
Une clé présente avec `null` est bien présente ; la règle interne décide du sort de cette valeur.

| Tableau reçu | Règle concernée | Chemin ou comportement |
|---|---|---|
| aucune clé `city` | présence obligatoire | `[city]`, code `MISSING_FIELD_ERROR` |
| `city: ''` | `NotBlank` interne | `[city]`, code de `NotBlank` |
| clé `comment` absente | `Optional` | aucune violation de présence |
| `comment` trop long | `Length` interne | `[comment]` |
| clé `debug` ajoutée | clé inconnue | `[debug]`, code `NO_SUCH_FIELD_ERROR` |

`allowMissingFields: true` permet les clés manquantes de manière globale, y compris celles normalement obligatoires.
Cette option peut convenir à un contrat partiel, mais elle change la sémantique de toute la structure.
`allowExtraFields: true` accepte les clés supplémentaires ; elle ne les supprime pas du tableau.
Le Validator observe les données sans les nettoyer ni les rendre immuables.
Ne pas valider permissivement puis transmettre toutes les clés à un mécanisme d'écriture non contrôlé.

`Collection` accepte `null` sans violation par elle-même ; utiliser une règle de présence si la structure est obligatoire.
Un tableau vide n'est pas `null` : il peut échouer parce que ses clés requises manquent.
Pour une valeur de type inconnu, contrôler d'abord le type afin d'obtenir un diagnostic lisible.
Les chemins des clés utilisent des crochets ; les chemins des propriétés d'un DTO utilisent la notation pointée.
Une structure imbriquée peut produire `[address][city]`, distinct de `address.city`.

## 2. Sequentially protège une chaîne de règles sur la même valeur

Un tableau de contraintes ordinaires ne demande pas au Validator d'arrêter à la première erreur.
`Sequentially` applique ses contraintes dans l'ordre et s'arrête dès qu'une étape ajoute une violation.
Il reçoit toujours la même valeur ; une étape ne la transforme pas pour la suivante.
Ce mécanisme permet de contrôler présence, type puis forme avant une vérification plus spécifique.
Il ne remplace pas `GroupSequence`, qui ordonne des groupes de validation, potentiellement sur plusieurs propriétés.

```php
$checks = new Assert\Sequentially([
    new Assert\NotBlank(),
    new Assert\Type('string'),
    new Assert\Length(min: 4),
]);
```

Avec cette composition, une valeur entière non vide échoue sur le type avant d'atteindre la longueur.
Une chaîne trop courte arrive jusqu'à la troisième étape.
Si une étape composite ajoute plusieurs violations, l'arrêt ne les réduit pas magiquement à une seule.
Les groupes sélectionnés restent pertinents : une règle exclue du groupe courant ne protège rien.
Utiliser un callback instrumenté permet de prouver qu'une vérification coûteuse n'a pas été appelée.

## 3. AtLeastOneOf exprime une alternative, pas un catalogue d'erreurs

`AtLeastOneOf` accepte la valeur dès qu'une de ses contraintes internes réussit.
Si toutes échouent, elle produit sa violation globale avec le code `AT_LEAST_ONE_OF_ERROR`.
`includeInternalMessages` contrôle l'ajout de détails internes au message final.
Les violations de chaque branche ne deviennent pas une liste de violations indépendante à exposer telle quelle.
Cette différence compte pour un client qui s'appuie sur un code d'erreur stable.

Pour « email ou identifiant interne », deux formes alternatives peuvent être utiles.
Mais une contrainte de format comme `Email` accepte le vide : cette branche peut rendre un vide accepté par toute l'alternative.
Placer la règle de présence hors de l'alternative, et ordonner les contrôles si le type n'est pas garanti.
Deux branches valides ne produisent pas deux succès à compter ; le résultat reste simplement sans violation.

| Besoin | Composition | Question à tester |
|---|---|---|
| toutes les règles doivent réussir | tableau de contraintes | plusieurs violations sont-elles attendues ? |
| arrêter après l'étape invalide | `Sequentially` | la dernière règle a-t-elle été appelée ? |
| une règle suffit | `AtLeastOneOf` | le vide passe-t-il par une branche permissive ? |
| première phase entière avant la suivante | `GroupSequence` | quel groupe bloque la progression ? |

Éviter de dépendre du texte complet d'une alternative : messages, options et traduction peuvent le faire varier.
Vérifier la validité, le chemin, le code et les effets d'une règle instrumentée fournit un contrat plus stable.
Une règle qui appelle un service doit rester sans effet métier irréversible : valider peut être répété.

## Observer dans l'application

```bash
symfony console debug:validator 'App\Validation\ImportInput'
symfony console debug:autowiring ValidatorInterface
php bin/phpunit tests/Validation
```

La commande de métadonnées décrit une classe existante, pas une contrainte `Collection` créée uniquement dans un test.
Pour cette dernière, appeler directement le Validator du conteneur et relever chaque chemin/code.
Un `KernelTestCase` de service vérifie les données qu'un vrai appelant remet à cette frontière.
Dans un test HTTP complémentaire, laisser le mapping désérialiser et vérifier la liste d'erreurs renvoyée.
Les données d'exemple ne sont pas à journaliser automatiquement dans l'application réelle.

Sources : [Collection 8.0](https://symfony.com/doc/8.0/reference/constraints/Collection.html), [Sequentially 8.0](https://symfony.com/doc/8.0/reference/constraints/Sequentially.html), [AtLeastOneOf 8.0](https://symfony.com/doc/8.0/reference/constraints/AtLeastOneOf.html).
Lecture locale : `vendor/symfony/validator/Constraints/CollectionValidator.php`, `SequentiallyValidator.php` et `AtLeastOneOfValidator.php`.
Comparer les méthodes `validate()` permet de voir où le chemin change, où la boucle s'arrête et où la violation globale se construit.
