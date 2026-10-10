# Exercices — Module 09

Code dans `src/TwigForms/` (namespace `App\TwigForms`), templates de test dans `templates/twig_forms/` (ou chargés via `ArrayLoader` dans les tests), tests dans `tests/TwigForms/` (namespace `App\Tests\TwigForms`) : `php bin/phpunit tests/TwigForms`. Écris le test d'abord.

## Partie Twig

### 1. Héritage et `parent()`
Crée un layout `base` avec des blocs `title` et `body`, et deux templates enfants. L'un surcharge `title` en réutilisant `{{ parent() }}`.
- Test : rendre avec `Twig\Environment` + `ArrayLoader` (templates en chaînes) et comparer la sortie.
- Question : que se passe-t-il d'un contenu placé hors `block` dans un template enfant ?

### 2. `include` vs `embed` vs `use` vs `macro`
Écris un composant « carte » de quatre façons (include avec `only`, embed avec bloc surchargé, use, macro).
- Test : une assertion par variante ; prouve qu'une macro **ne voit pas** une variable du contexte appelant (avec `strict_variables`).
- Réflexion : pour quel besoin choisirais-tu chacune ?

### 3. Extension Twig : filtre + fonction + test
Crée `App\TwigForms\MoneyExtension` (classe `AbstractExtension`) avec un filtre `money` (formatage), une fonction `initials(name)` et un test `is_even`. Puis refais le filtre avec l'attribut `#[AsTwigFilter]` dans un autre service.
- Critère : `bin/console debug:twig --filter=money` les affiche ; test d'intégration via le service `twig`.
- Question : quand utiliser une *runtime extension* plutôt qu'une extension classique ?

### 4. Échappement et XSS
Écris un filtre `highlight` qui entoure un mot de `<mark>` et renvoie du HTML.
- Tests : sans `is_safe`, la sortie est échappée ; avec `is_safe: ['html']`, non. Montre le cas dangereux (entrée utilisateur contenant `<script>`) et corrige-le (échapper l'entrée **dans** le filtre).
- Question : quelle différence entre `|raw`, `|e('html_attr')` et `{% autoescape false %}` ?

## Partie Forms

### 5. FormType testé
Crée `Task` (title, dueDate, done) avec contraintes `Assert`, et `TaskType`.
- Test avec `TypeTestCase` : soumission valide ⇒ objet hydraté ; titre vide ⇒ formulaire invalide avec 1 erreur ; options (`data_class`) correctes.
- Question : pourquoi `required: true` ne remplace-t-il pas `NotBlank` ?

### 6. Événements : champ dynamique
Dans `ProductType`, ajoute un champ `sku` seulement si le `Product` n'a pas encore d'identifiant (`PRE_SET_DATA`), et un événement `PRE_SUBMIT` qui normalise le titre (trim + lowercase) avant soumission.
- Tests : création ⇒ présence de `sku` ; édition ⇒ absence ; titre `"  Foo "` ⇒ `"foo"`.
- Réflexion : pourquoi écouter `POST_SUBMIT` du champ parent (et non du formulaire racine) pour un champ ville dépendant d'un pays ?

### 7. DataTransformer + `inherit_data`
(a) Crée `TagsTransformer` (string `"a, b"` ↔ `['a','b']`) et branche-le sur un champ. Test : transform/reverseTransform, `null`, chaîne vide, et un cas qui lève `TransformationFailedException`.
(b) Crée `AddressType` avec `inherit_data` et réutilise-le dans deux formulaires dont les objets n'ont aucune classe commune.
- Réflexion : model transformer ou view transformer pour (a) ? Que vaut `isSynchronized()` après une `TransformationFailedException` ?

### 8. CollectionType et thème
Formulaire `Invoice` contenant une collection de `LineType` (`allow_add`, `allow_delete`, `by_reference: false`, `delete_empty`). Écris un thème de formulaire minimal qui change le rendu de `form_row` et applique-le (config globale **ou** `{% form_theme %}`).
- Tests : soumettre 3 lignes puis 2 ⇒ la 3e est retirée, `removeLine()` est appelé ; le rendu contient ta classe CSS. Observe le `data-prototype` / `form.vars.prototype`.
- Question : que se passe-t-il avec `by_reference: true` pour une relation Doctrine inverse ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.

## Critères complémentaires pour les exercices 1–8

Appliquer la [méthode commune](../README.md). **Parcours A : 1–4 puis 9. Parcours B : 5–8 puis 10–12.** Le défi final relie les deux.

| Exercice | Acceptation supplémentaire |
|---|---|
| 1 | Comparer titre et corps exacts ; texte rendu hors bloc dans l'enfant → erreur de syntaxe, pas hypothèse de contenu silencieusement ignoré. |
| 2 | Macro sans variable locale transmise → erreur en mode strict ; ne pas confondre contexte local et globals Twig disponibles. |
| 3 | Ne pas enregistrer simultanément deux filtres `money` concurrents pour prouver leur résultat ; tester chaque voie puis comparer la sortie `12,50 €` selon le format choisi. |
| 4 | Avec une entrée `<script>`, aucun élément script dans le DOM rendu, même quand le filtre est déclaré sûr ; conserver le balisage `mark` voulu. |
| 5 | Brancher `ValidatorExtension` dans `TypeTestCase` pour exercer `NotBlank` ; vérifier `getErrors(true)` et la donnée hydratée, pas uniquement `isSynchronized()`. |
| 6 | Nouvelle instance de formulaire pour chaque soumission ; on ne soumet pas deux fois le même Form. |
| 7 | Transformer model : model ↔ norm ; erreur de transformation → champ désynchronisé et aucune utilisation métier de sa valeur. |
| 8 | Deux constructions/soumissions distinctes pour passer de 3 à 2 lignes ; nom de la ligne retirée vérifié dans `removeLine()`. Pas de Doctrine nécessaire : objets PHP et collection contrôlée. |

## 9. Parcours A guidé — HTML observé par un navigateur

Créer une page de liste et un fragment de carte à contexte explicite. Chaque titre mène à une route nommée et passe par l'échappement par défaut.

- Acceptation : deux fixtures → deux liens avec URLs générées ; titre `<b>Ada</b>` affiché comme texte, aucun élément `b` injecté ; variable obligatoire absente → erreur avec `strict_variables`.
- Test de rendu au niveau Twig puis test HTTP avec assertions DOM pour confirmer l'intégration.
- Indice : passer uniquement les variables du fragment, puis comparer `textContent` et structure HTML ; ne pas utiliser `raw` pour obtenir une égalité de chaîne.

## 10. Parcours B guidé — cycle HTTP et CSRF

Exposer `TaskType` par un contrôleur. Choisir un identifiant CSRF de session `lesson_task`, hors liste sans état de la recette. Le service métier ne doit être appelé que sur succès.

- Acceptation : GET → 200 et formulaire ; POST titre vide avec token valable → 422 et erreur ; token altéré → 422 sans effet ; POST valide → 303 puis GET 200.
- Récupérer et soumettre le formulaire avec le même client ; compter un seul appel métier. Le test ne doit pas générer lui-même un token en contournant le formulaire rendu.
- Indice : passer le Form à `render()` pour bénéficier du statut d'erreur ; la redirection 303 est un choix explicite du contrôleur.

## 11. Parcours B intermédiaire — upload et conversion impossible

Ajouter un champ fichier texte facultatif non mappé, limité à 1 Mo, puis un champ entier avec un cas de transformation impossible.

- Acceptation : petit fichier `.txt` de contenu texte accepté ; fichier de 2 Mo refusé ; contenu incompatible avec l'extension refusé ; champ entier `"abc"` → désynchronisé et invalide ; aucun fichier déplacé sur formulaire invalide.
- Sur succès, vérifier le contenu stocké dans un dossier temporaire et un nom choisi par le serveur ; nettoyer après test. Une erreur d'E/S doit être observable, sans afficher de chemin interne au visiteur.
- Indice : `UploadedFile` en mode test pour la fixture ; soumission multipart pour le test HTTP, pas uniquement un appel au transformer.

## 12. Parcours B avancé — option et extension de type

Créer une extension de `TextType` apportant `help_icon: null|string`, la transférer à la vue et la rendre dans un thème, sans désactiver l'échappement.

- Acceptation : option absente → aucun pictogramme ; `help_icon: '?'` → élément attendu ; valeur tableau → `InvalidOptionsException` à la création ; option mal orthographiée → `UndefinedOptionsException` ; le champ reste soumettable.
- Prouver avec `debug:form` que l'extension s'applique et que la valeur n'existe dans le HTML que grâce au thème.
- Indice : OptionsResolver configure le type, `buildView()` prépare la vue, le thème choisit le balisage : trois responsabilités distinctes.

## Défi final — dépôt de demande avec pièce jointe

Assembler formulaire, validation, transformer, upload et page récapitulative Twig. Utiliser un service de test pour enregistrer les appels et un dossier temporaire ; aucune couche Doctrine n'est imposée.

Livrables : tests de rendu et de type, plus le scénario HTTP complet. Acceptation : erreurs visibles au bon champ, saisie conservée après échec, aucune action métier si CSRF/conversion/validation échoue, fichier traité seulement après succès, redirection puis récapitulatif échappé. Inclure un titre ressemblant à du HTML et un nom de fichier trompeur dans les fixtures. Indice final : d'abord le cycle GET/POST minimal, ensuite transformer et fichier ; garder séparés le diagnostic interne et les messages destinés au visiteur.
