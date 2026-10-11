# Approfondissement — soumission partielle et erreurs visibles

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Cette fiche prolonge le cycle de formulaire par trois distinctions : champ absent, erreur rattachée et valeur Twig vide.
Elle utilise le socle **Symfony 8.0** et **Twig 3.x**, vérifié dans Symfony **8.1.8** et Twig installé.
Prérequis : `submit()`, transformations, Validator, rendu `form_row()` et échappement.
Les exemples concernent des objets PHP en mémoire ; aucune intégration Doctrine n'est nécessaire.

## 1. Une soumission partielle ne vide pas nécessairement les champs absents

`Form::submit($data, $clearMissing)` prend `true` comme seconde valeur par défaut.
Sur un formulaire composé, un enfant absent est alors soumis avec `null`.
Ses transformers et son `empty_data` déterminent ensuite la valeur effectivement écrite : pas toujours un `null` final.
Avec `false`, seuls les enfants présents sont soumis et les autres conservent leurs données.
Le request handler emploie ce comportement pour la méthode HTTP `PATCH`.
Le nom `PATCH` ne choisit pas à lui seul les champs modifiables : le FormType définit toujours leur liste.

```php
// Fragment d'expérience au niveau Form, avec un formulaire neuf à chaque appel.
$form->submit(['nickname' => 'Lin'], clearMissing: false);
$submitted = $form->get('nickname')->isSubmitted();
$untouched = $form->get('city')->isSubmitted();
```

Ici `nickname` est soumis et `city` ne l'est pas, si ces deux champs existent dans le type.
Une chaîne vide présente n'est pas un champ absent : elle suit les règles normales de transformation et de validation.
Un champ `disabled` ignore la donnée envoyée ; `mapped: false` reçoit la donnée sans l'écrire dans l'objet associé.
Un champ inconnu appartient aux données supplémentaires et rend normalement le formulaire invalide.
Ces trois options répondent donc à trois questions différentes.

| Entrée et configuration | Effet attendu | Observation utile |
|---|---|---|
| champ omis, `clearMissing: false` | ancienne valeur conservée | enfant non soumis |
| champ omis, `clearMissing: true` | soumission d'une valeur absente | valeur finale après `empty_data` |
| champ présent avec `''` | traitement du vide | contraintes et transformations |
| champ désactivé falsifié | valeur ignorée | ancienne donnée conservée |
| champ non mappé | accessible au formulaire | objet métier inchangé |
| champ inconnu | erreur de données supplémentaires | `getExtraData()` et erreurs racine |

### Validation partielle et invariant global

Les contraintes portées par les champs non soumis ne sont pas un contrôle complet de toutes les anciennes données.
En revanche, les contraintes de l'objet mappé peuvent encore examiner plusieurs propriétés pendant sa validation.
Ne conclure donc ni « tout est revalidé » ni « seules ces clés peuvent produire une erreur » sans tester le montage.
Un objet dont `start` et `end` doivent rester cohérents exige un invariant global explicite.
Le formulaire peut avoir modifié l'objet avant que `isValid()` ne soit consulté.
Pour garder un état métier intact en cas d'échec, utiliser un objet d'entrée séparé et appliquer seulement le résultat valide.

## 2. Une erreur possède un lieu de validation et un lieu de rendu

Le Validator désigne une propriété par un chemin ; Form traduit ce chemin vers un enfant de son arbre.
Cette traduction peut échouer si le formulaire ne contient pas de champ équivalent.
`error_mapping` permet alors de choisir un champ visible pour une violation métier.
Il associe un chemin de violation à un nom de champ, sans changer la contrainte ni sa logique.
`error_bubbling` règle la remontée d'une erreur vers le parent ; il ne décide pas si la donnée est valide.

```php
// Fragment d'options d'un formulaire composé.
$options = [
    'error_mapping' => ['deliveryWindow' => 'deliveryDate'],
];
```

Dans cet exemple, le chemin métier `deliveryWindow` doit être réellement produit par la validation.
Une erreur déjà rattachée correctement n'a pas besoin d'une correspondance supplémentaire.
La clé spéciale `.` peut servir de destination générale pour des violations non mappées ; la réserver à une intention claire.
Faire remonter toutes les erreurs masque parfois le champ que l'utilisateur doit corriger.
Comparer `getErrors(false)` sur la racine, sur l'enfant, puis `getErrors(true)` sur tout l'arbre.

| Symptôme | Hypothèse à vérifier | Preuve attendue |
|---|---|---|
| formulaire invalide sans message près du champ | erreur sur parent ou enfant non rendu | arbre des erreurs et DOM |
| message doublé | racine et enfant rendus deux fois | nombre d'occurrences dans le HTML |
| erreur au mauvais champ | mapping ou bubbling inadapté | `FormError::getOrigin()` |
| message visible mais succès métier déclenché | contrôle du résultat absent | compteur d'appels métier |

Le thème décide de l'emplacement HTML, pas de la validité de l'objet.
Tester le rendu final protège l'accessibilité de l'erreur mieux qu'un simple `assertFalse($form->isValid())`.
Conserver l'échappement du texte saisi et du message ; `raw` ne répare pas un mauvais mapping.

## 3. Le fallback Twig peut effacer un faux explicite

`default` s'applique aux valeurs considérées vides par Twig, dont `false` et la chaîne vide.
L'opérateur `??` s'applique à l'absence ou à `null`, en conservant les autres valeurs.
Ainsi une case non cochée peut être remplacée à tort par une valeur vraie si le template utilise un fallback trop large.
Le nombre zéro, la chaîne vide et le booléen faux méritent chacun leur fixture de rendu.

```twig
{{ label|default('Sans titre') }}
{{ label ?? 'Non renseigné' }}
```

Avec `label = ''`, ces expressions ne donnent pas le même résultat.
Pour un booléen, rendre un texte explicite avec une condition évite de confondre sa représentation vide avec son absence.
Le choix dépend du contrat d'affichage ; il ne remplace pas les contraintes du formulaire.

## Observer et vérifier

```bash
symfony console debug:form 'App\TwigForms\ProfileType'
symfony console debug:form 'App\TwigForms\ProfileType' error_mapping
symfony console lint:twig templates/
php bin/phpunit tests/TwigForms
```

Adapter le nom à la classe créée ; utiliser une nouvelle instance de Form pour chaque soumission.
`KernelTestCase` avec `form.factory` exerce les extensions réelles ; `WebTestCase` vérifie statut et HTML.
Avec CSRF activé, commencer par le formulaire rendu et conserver le client entre lecture et envoi.
Lire `var/log/dev.log` devant une exception ; aucun profiler n'est présupposé.

Sources : [FormType 8.0](https://symfony.com/doc/8.0/reference/forms/types/form.html), [soumission directe 8.0](https://symfony.com/doc/8.0/form/direct_submit.html), [default de Twig 3.x](https://twig.symfony.com/doc/3.x/filters/default.html).
Code local : `Form.php`, `Extension/HttpFoundation/HttpFoundationRequestHandler.php`, `Extension/Validator/Constraints/FormValidator.php` sous `vendor/symfony/form/`.
Les ateliers évitent les changements de gestion des cases décochées propres à Form 8.1 : les valeurs présentes sont toujours explicites.
