# Module 09 — Twig & Forms

[Parcours](../README.md) · [Exercices](exercices.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

## Objectifs, prérequis et repères

> Application : **Symfony 8.1.8, PHP ≥ 8.4**. La [certification Symfony](https://certification.symfony.com/exams/symfony.html) cible **8.0** : les mentions **8.1 uniquement** sont des compléments pour cette application, pas des acquis à attribuer à 8.0.

Ce dossier contient **deux parcours distincts** : **A — Twig** (sections 1–5, exercices 1–4 puis 9) pour composer et échapper le HTML ; **B — Forms** (sections 6–11, exercices 5–8 puis 10–12) pour transformer et valider une soumission. Le défi final les rassemble.

Prérequis Twig : contrôleur, Response et [routage](../11-routing/README.md). Prérequis Forms : parcours Twig, [Validator](../12-validator/README.md), PropertyAccess (module 10) et CSRF (module 06). Objectifs : expliquer le HTML final, suivre GET → POST → erreurs ou redirection, distinguer transformation/validation et étendre proprement les types. Aucun besoin de Doctrine.

Applique la [méthode de travail commune](../README.md) : prédire, exécuter, observer, expliquer. Les exemples ci-dessous illustrent un mécanisme ; les exercices restent à réaliser. Le profiler n'est pas installé : utiliser les commandes et les tests proposés. Son installation facultative passerait par `composer require --dev symfony/profiler-pack`.

Deux composants très présents à la certification. `symfony/twig-bundle` (Twig 3.30 installé) et `symfony/form` sont installés dans cette app (`symfony/twig-bridge` fournit la glue : fonctions `form_*`, `path()`, `asset()` si `symfony/asset` est installé...).

# Partie A — Twig

## 1. Héritage : `extends`, `block`, `parent()`

```twig
{# templates/base.html.twig #}
<title>{% block title %}Mon site{% endblock %}</title>
<main>{% block body %}{% endblock %}</main>

{# templates/page.html.twig #}
{% extends 'base.html.twig' %}
{% block title %}{{ parent() }} — Page{% endblock %}   {# parent() = contenu du bloc parent #}
{% block body %}Hello {{ name }}{% endblock %}
```
- `{% extends %}` doit être la **première** balise du template enfant ; le contenu rendu hors d'un `block` dans un template enfant provoque une erreur de syntaxe (les déclarations comme `set` restent possibles).
- `{% extends expr %}` accepte une expression (ex. `{% extends ajax ? 'a.twig' : 'b.twig' %}`) ou un tableau de fallback.
- `{{ block('title') }}` affiche un bloc ailleurs ; `{% block title %}…{% endblock title %}` nommé à la fermeture ; raccourci `{% block title page_title %}`.

## 2. `include`, `embed`, `use`, `macro`

| Outil | Usage |
|-------|-------|
| `{% include 'x.twig' with {a: 1} only %}` ou `{{ include('x.twig', {a: 1}, with_context = false) }}` | inclure un fragment ; hérite du contexte sauf `only` |
| `{% embed 'card.twig' %}{% block body %}…{% endblock %}{% endembed %}` | **include + possibilité de surcharger les blocs** du template inclus |
| `{% use 'blocks.twig' %}` | réutilisation **horizontale** : importe des blocs (pas de rendu) |
| `{% macro input(name) %}…{% endmacro %}` + `{% import 'm.twig' as m %}` / `{% from 'm.twig' import input %}` | fonctions de template ; **n'accèdent pas** aux variables locales du template appelant (les globals Twig restent disponibles ; il faut passer les variables locales ; `_self` pour les macros du même fichier : `{% import _self as m %}`) |

- `include` ≠ `embed` : `embed` = `include` + blocs surchargeables.
- Un enfant qui `use` un template ne peut pas avoir de contenu hors blocs dans le template utilisé.
- `{% include %}` avec une liste : `{% include ['a.twig', 'b.twig'] %}` (premier existant) ; `ignore missing`.

## 3. Filtres, fonctions, tests, extensions

Voie classique :
```php
final class AppExtension extends AbstractExtension
{
    public function getFilters(): array   { return [new TwigFilter('price', $this->price(...), ['is_safe' => ['html']])]; }
    public function getFunctions(): array { return [new TwigFunction('now_label', [AppRuntime::class, 'label'])]; }
    public function getTests(): array     { return [new TwigTest('even', fn (int $n) => 0 === $n % 2)]; }
}
```
- **Extension runtime** (`RuntimeExtensionInterface`) : la fonction vise `[AppRuntime::class, 'label']` ; la classe runtime est un service **chargé paresseusement** (dépendances lourdes sans ralentir Twig).
- Voie moderne : attributs **`#[AsTwigFilter]`, `#[AsTwigFunction]`, `#[AsTwigTest]`** sur des méthodes de n'importe quel service (autoconfiguration vérifiée dans `TwigExtension` du twig-bundle) :
```php
final class Format
{
    #[AsTwigFilter('price', isSafe: ['html'])]
    public function price(float $v): string { return number_format($v, 2).' €'; }
}
```
- Un service implémentant `ExtensionInterface` est auto-tagué `twig.extension` (autoconfigure).
- Variables globales : `twig.globals` dans `config/packages/twig.yaml` ou `GlobalsInterface`.
- Variable `app` (`AppVariable`) : `app.user`, `app.request`, `app.session`, `app.environment`, `app.debug`, `app.flashes`.

## 4. Échappement (escaping)

- **Autoescape activé par défaut** : stratégie `html` pour les templates `.html.twig` (stratégie « par nom de fichier »). Le HTML est échappé, pas le texte brut.
- `{{ v|raw }}` désactive l'échappement ; `{{ v|e('js') }}`, `e('html_attr')`, `e('css')`, `e('url')` pour d'autres contextes.
- `{% autoescape 'js' %}…{% endautoescape %}` change la stratégie localement.
- **Double échappement** : un filtre « safe » (`is_safe: ['html']`) dit à Twig que sa sortie est déjà sûre → pas de ré-échappement. Marquer `is_safe` à tort ⇒ faille XSS. Le résultat d'un filtre non-safe est échappé à l'affichage.

## 5. Autres points utiles
- `strict_variables` : en test dans cette app (`when@test` de `twig.yaml`) — une variable inconnue lève une erreur au lieu d'être `null`.
- Cache des templates compilés dans `var/cache/<env>/twig` ; `cache:clear` reconstruit le cache ; `lint:twig` vérifie la syntaxe sans le vider.
- `{{ path('route') }}`, `{{ url() }}`, `{{ asset('…') }}` (nécessite `symfony/asset`), `{{ dump(x) }}` (selon les extensions de debug installées), `{{ form(form) }}`.
- `|default`, `|length`, `|join`, `|map`/`|filter`/`|reduce` (arrow functions), opérateur `??`, `is defined`, `is empty`, `in`, `matches`, `..`.

# Partie B — Forms

## 6. FormType de base

```php
final class TaskType extends AbstractType
{
    public function buildForm(FormBuilderInterface $builder, array $options): void
    {
        $builder->add('title', TextType::class)
                ->add('dueDate', DateType::class, ['widget' => 'single_text'])
                ->add('save', SubmitType::class);
    }
    public function configureOptions(OptionsResolver $resolver): void
    {
        $resolver->setDefaults(['data_class' => Task::class]);
    }
}

// contrôleur
$form = $this->createForm(TaskType::class, $task);
$form->handleRequest($request);
if ($form->isSubmitted() && $form->isValid()) { /* … */ return $this->redirectToRoute('…'); }
return $this->render('task/new.html.twig', ['form' => $form]);
```
- Méthodes d'un type : `buildForm`, `buildView`, `finishView`, `configureOptions`, `getParent`, `getBlockPrefix`.
- Avec `createForm(TaskType::class)`, le **nom** du formulaire (préfixe des champs HTML, ex. `task[title]`) est le **block prefix** du type (`task`, dérivé du nom de classe sans `Type`) ; `createNamed()`/`createNamedBuilder()` pour en choisir un autre.
- Trois représentations des données : **model** (objet/entité), **norm** (normalisée), **view** (ce que montre le HTML, souvent des strings). Les transformers passent de l'une à l'autre.
- `mapped: false` : champ non lié à l'objet ; `property_path`, `by_reference`, `empty_data`, `required` (HTML seulement, pas validation), `constraints`, `attr`, `label`, `help`, `disabled`.
- CSRF : activé par défaut par `FormTypeCsrfExtension` (champ `_token`, options `csrf_protection`, `csrf_field_name`, `csrf_token_id`).
- Les groupes de validation : option `validation_groups` ; contraintes sur la classe via `#[Assert\...]`.

## 7. Événements de formulaire

Ordre au **chargement** : `PRE_SET_DATA` → `POST_SET_DATA`. À la **soumission** : `PRE_SUBMIT` → `SUBMIT` → `POST_SUBMIT`.

| Événement (`FormEvents::`) | Classe | Moment / usage (d'après `vendor/symfony/form/FormEvents.php`) |
|---|---|---|
| `PRE_SET_DATA` | `PreSetDataEvent` | début de `setData()` : modifier les données de pré-remplissage, **ajouter/retirer des champs selon l'objet** |
| `POST_SET_DATA` | `PostSetDataEvent` | fin de `setData()` : état final dans les 3 représentations |
| `PRE_SUBMIT` | `PreSubmitEvent` | début de `submit()` : modifier les données **brutes de la requête**, ajouter/retirer des champs |
| `SUBMIT` | `SubmitEvent` | après transformation vers *norm*, avant retour vers model/view ; on peut encore ajouter/retirer des champs |
| `POST_SUBMIT` | `PostSubmitEvent` | tout à la fin ; lire les données dénormalisées (tester `isSynchronized()` avant) |

```php
$builder->addEventListener(FormEvents::PRE_SET_DATA, function (PreSetDataEvent $event): void {
    $product = $event->getData();
    if (null === $product?->getId()) {
        $event->getForm()->add('sku', TextType::class);   // champ seulement à la création
    }
});
```
- Pour les **champs dépendants** (pays → ville) : écouter `POST_SUBMIT` sur le champ parent (pas sur le formulaire racine) car les données y sont dénormalisées.
- Un `EventSubscriberInterface` via `addEventSubscriber()` regroupe plusieurs événements.
- Les classes `Event\*Event` sont les alias de `FormEvents::ALIASES` (utilisables comme nom d'événement).

## 8. DataTransformer

```php
final class TagsTransformer implements DataTransformerInterface
{
    public function transform(mixed $tags): mixed         // MODEL -> NORM (puis les view transformers produisent VIEW)
    { return implode(', ', $tags ?? []); }

    public function reverseTransform(mixed $string): mixed // NORM -> MODEL (après les view transformers)
    {
        if (!$string) { return []; }
        return array_map('trim', explode(',', $string));   // en cas d'erreur : throw new TransformationFailedException('…');
    }
}
$builder->get('tags')->addModelTransformer(new TagsTransformer());
```
- `addModelTransformer` (model ↔ norm) vs `addViewTransformer` (norm ↔ view). `CallbackTransformer` pour du simple.
- `transform(null)` doit être géré ; `reverseTransform('')` renvoie souvent `null`.
- `TransformationFailedException` ⇒ le champ devient **non synchronisé** (`isSynchronized() === false`) et une erreur est ajoutée ; `invalid_message` personnalise le texte.
- DataTransformer ≠ validation : le transformer convertit, les contraintes valident.

## 9. `inherit_data` et formulaires composés

`inherit_data: true` : le sous-formulaire reçoit les **mêmes données que son parent** (pas de sous-objet) → réutiliser un groupe de champs (ex. `AddressType`) sur plusieurs classes sans propriété commune.
```php
$builder->add('address', AddressType::class, ['inherit_data' => true]);
```
Chaque champ enfant se mappe alors directement sur l'objet parent (ajuste avec `property_path` si les noms diffèrent). Détail vérifié dans `FormType` : `error_bubbling` vaut par défaut `compound && !inherit_data`.

## 10. CollectionType

```php
$builder->add('tags', CollectionType::class, [
    'entry_type'    => TagType::class,
    'entry_options' => ['label' => false],
    'allow_add'     => true,       // accepte de nouveaux éléments (prototype)
    'allow_delete'  => true,       // retire les éléments absents du POST
    'by_reference'  => false,      // force addTag()/removeTag() sur l'objet
    'delete_empty'  => true,       // bool ou callable
    'prototype'     => true,       // data-prototype pour le JS
]);
```
- `allow_add` / `allow_delete` autorisent les modifications à la soumission ; pour une interface dynamique côté navigateur, ajouter du **JavaScript** pour créer/supprimer les lignes (`form.vars.prototype`, `__name__`).
- `by_reference: false` est nécessaire pour que les `addX()`/`removeX()` du modèle soient appelés (sinon une collection objet peut être modifiée sur place sans appel des méthodes du parent ; observer ce comportement avec tes objets PHP).
- `keep_as_list` (option vérifiée) garde les index séquentiels.

## 11. Thèmes de formulaire

- Thème global par défaut : `form_div_layout.html.twig`. Config : `twig.form_themes: ['bootstrap_5_layout.html.twig']` (Twig bridge fournit aussi `bootstrap_3/4/5`, `foundation_5/6`, `tailwind_2`, `form_table_layout`, `daisyui_5`…).
- Thème par template : `{% form_theme form 'form/theme.html.twig' %}` (ou `with [...]`, `only`).
- Blocs : `form_row`, `form_widget`, `form_label`, `form_errors`, `form_help` ; les noms de bloc se composent du **block prefix** : `_task_title_widget`, `text_widget`, `form_widget`. Un thème est un template avec des `{% block %}` qui surchargent par héritage ; fallback sur le parent du type (`getParent`).
- Rendu manuel : `{{ form_start(form) }} … {{ form_row(form.title) }} … {{ form_end(form) }}` ; `form_end` rend les champs restants + `_token` (sauf `{{ form_end(form, {render_rest: false}) }}`).
- Variable `form.vars` (`value`, `errors`, `id`, `full_name`, `attr`, `block_prefixes`…).

## Ce qui change vs Symfony 6/7
- `renderForm()` n'existe plus (supprimé en 7.0) : `$this->render('…', ['form' => $form])` convertit le form en vue et renvoie **422** si le formulaire soumis est invalide (vérifié dans `AbstractController::render`).
- Form 8.0 : `UrlType::default_protocol` passe de `'http'` à `null` ; `validation.xml` du composant supprimé ; `ResizeFormListener::preSetData()` supprimé (utiliser `postSetData()`).
- Form 8.1 : support des cases décochées dans les request handlers, `labels` pour `DateType`, `min`/`max` par défaut pour `BirthdayType` single_text. `FormFlow` (`createFormFlowBuilder()`, `NavigatorFlowType`, `ResetFlowType`) pour les formulaires multi-étapes (ajouté en 7.4, `CHANGELOG` du composant ; hors périmètre du quiz).
- Twig : `#[AsTwigFilter]`/`#[AsTwigFunction]`/`#[AsTwigTest]` sont autoconfigurés par le twig-bundle (tu n'as plus besoin d'une classe `AbstractExtension` pour un filtre simple).
- `input=date_point` (7.4) pour DateType/TimeType/DateTimeType.

## Piège certification
- `required: true` ne valide **pas** côté serveur (attribut HTML `required` seulement) : ajoute `NotBlank`.
- `handleRequest` ne soumet que si la méthode HTTP correspond (`POST` par défaut) ; `isSubmitted()` avant `isValid()`.
- `PRE_SUBMIT` : données **brutes** (tableau) ; `POST_SUBMIT` : données dénormalisées. Ajouter un champ dynamique dépendant d'un autre → `POST_SUBMIT` du champ parent.
- `embed` (blocs surchargeables) ≠ `include` ≠ `use` (blocs importés) ≠ `macro` (pas de contexte).
- `{{ x|raw }}` d'une entrée utilisateur = XSS ; `is_safe` sur un filtre déclare sa sortie sûre.
- `inherit_data` ≠ `mapped: false` ; `by_reference: false` pour déclencher `addX/removeX`.
- Une macro **n'a pas accès** aux variables du template appelant.

## Observer dans l'app
- `bin/console debug:twig` (filtres, fonctions, tests, globals, extensions) ; `debug:twig --filter=price`.
- `bin/console lint:twig templates/`.
- `bin/console debug:form TaskType` (options, parent, extensions) ; `debug:form --show-deprecated` ; `debug:form TaskType required` (le deuxième argument est une option, pas un champ).
- `bin/console debug:container --tag=twig.extension` ; `debug:config twig` ; `debug:autowiring Form`.
- Après installation facultative du profiler : profiler : panneau **Form** (arbre, données submittées/modèle/vue, erreurs) et **Twig** (templates rendus, temps).
- Tests : `TypeTestCase` pour un FormType (`$this->factory->create(...)`, `->submit([...])`, `isSynchronized()`) ; templates : test KernelTestCase avec `self::getContainer()->get('twig')->render(...)` ; `php bin/phpunit tests/TwigForms`.

## Doc officielle
- https://symfony.com/doc/current/templates.html
- https://twig.symfony.com/doc/3.x/ (balises `extends`, `embed`, `use`, `macro`, filtre `escape`)
- https://symfony.com/doc/current/templates.html#writing-a-twig-extension
- https://symfony.com/doc/current/forms.html
- https://symfony.com/doc/current/form/events.html
- https://symfony.com/doc/current/form/data_transformers.html
- https://symfony.com/doc/current/form/embedded.html (et `inherit_data` : https://symfony.com/doc/current/form/inherit_data_option.html)
- https://symfony.com/doc/current/form/form_collections.html
- https://symfony.com/doc/current/form/form_themes.html
- https://symfony.com/doc/current/form/unit_testing.html

## 12. Parcours B : suivre une soumission jusqu'à la réponse

`handleRequest()` reconnaît la méthode configurée et le nom du formulaire dans la requête. Un POST contenant `title=x` ne soumet pas nécessairement un formulaire nommé `task` qui attend `task[title]=x`. Pour un GET initial, `isSubmitted()` vaut `false` ; appeler `isValid()` avant soumission n'est pas le bon test d'état.

Le navigateur envoie des chaînes et éventuellement des fichiers. Les view transformers convertissent view → norm, les model transformers norm → model ; le mapping renseigne l'objet ; la validation examine les données synchronisées. Si une conversion échoue, le champ porte une erreur et `isSynchronized()` vaut `false`. Une valeur bien convertie peut néanmoins enfreindre une contrainte et rendre `isValid()` faux. Ne pas effectuer l'action métier dès que l'objet semble rempli.

| Situation | État attendu | Réponse du scénario |
|---|---|---|
| GET initial | non soumis | 200, formulaire et token présents |
| POST avec conversion impossible | soumis, champ désynchronisé | 422 avec erreurs en passant le Form à `render()` |
| POST avec titre vide | soumis, conversion possible, invalide | 422, erreur `NotBlank` |
| POST valide | soumis, synchronisé, valide | redirection 303 choisie, puis GET 200 |

La redirection après succès évite qu'un rafraîchissement de la page finale répète directement le POST. Elle ne fournit pas une garantie générale d'idempotence. L'état en mémoire d'un service n'est pas conservé entre requêtes PHP : vérifier l'appel à une dépendance ou un effet de test adapté sans inventer de persistence.

Pour CSRF, `form_end()` rend les champs restants dont le token ; si `render_rest: false`, rendre ce champ explicitement. La recette locale configure `submit` sans état : voir module 06. Pour un test isolant le token en session, fixer `csrf_token_id: lesson_task` et conserver le même client entre GET et POST. `TypeTestCase` assemble un système de formulaires minimal : ajouter `ValidatorExtension` pour que `NotBlank` soit exécuté ; il ne prouve pas la configuration CSRF et HTTP de l'application. Cette dernière se teste avec `WebTestCase` et une vraie soumission du formulaire récupéré.

### Upload et frontière de confiance

```php
use Symfony\Component\Form\Extension\Core\Type\FileType;
use Symfony\Component\Validator\Constraints as Assert;

$builder->add('attachment', FileType::class, [
    'mapped' => false,
    'required' => false,
    'constraints' => [new Assert\File(maxSize: '1M', extensions: ['txt'])],
]);
```

Le fichier est un `UploadedFile`, pas un nom fourni par le client à réutiliser aveuglément. `form_start()` adapte l'encodage en `multipart/form-data`. Après validation seulement, un service choisit un nom de stockage et déplace le fichier. `mapped: false` signifie qu'il faut le récupérer depuis le champ ; cela **ne supprime pas** ses contraintes. Taille, extension et type MIME doivent être cohérents. Un échec d'écriture peut encore survenir après validation : le présenter comme une erreur d'opération, pas comme une preuve que le fichier était valide sur le disque cible.

## 13. Type, extension de type et OptionsResolver

Un `AbstractType` décrit une composition réutilisable. Un `AbstractTypeExtension` enrichit tous les types qu'il déclare étendre, sans copier chaque formulaire. Pour une option `help_icon` sur `TextType`, une extension peut déclarer :

```php
public static function getExtendedTypes(): iterable
{
    return [TextType::class];
}

public function configureOptions(OptionsResolver $resolver): void
{
    $resolver->setDefault('help_icon', null);
    $resolver->setAllowedTypes('help_icon', ['null', 'string']);
}
```

L'extension est autoconfigurée via `form.type_extension`. `buildView()` transfère ensuite une option dans `$view->vars` pour un thème Twig ; une option PHP n'apparaît pas magiquement dans le HTML. `setRequired()` impose une option, `setAllowedValues()` restreint les valeurs, `setNormalizer()` transforme une valeur résolue. Une option inconnue ou mal typée provoque une exception de configuration **avant** la soumission ; ce n'est pas une erreur de saisie à afficher à l'utilisateur. Observer avec `debug:form 'App\TwigForms\TaskType'` et des tests de création du formulaire.

## 14. Parcours A : vérifier le contexte et le HTML produit

Pour un fragment affichant `name`, `include(..., with_context: false)` exige que `name` soit transmis explicitement. `strict_variables` fait remonter l'oubli au lieu de produire une chaîne vide. Une macro ne reçoit pas les variables locales du template appelant implicitement, mais les variables globales Twig restent disponibles : « pas de contexte local » ne signifie pas « aucune globale ».

```twig
<a href="{{ path('app_task_show', {id: task.id}) }}">{{ task.title }}</a>
```

L'URL est produite par le routeur ; le titre est échappé pour le HTML. `raw` ne valide pas la valeur et `is_safe` ne nettoie aucun contenu. Un test utile rend le titre `<b>Ada</b>` et vérifie le texte échappé, l'absence d'un élément `<b>` injecté, et le lien attendu. Garder la logique métier dans le service et fournir au template des données explicites. Référence : [formulaires 8.0](https://symfony.com/doc/8.0/forms.html) ; source locale : `Form.php`, `HttpFoundationRequestHandler.php`, `AbstractController::render()`.
