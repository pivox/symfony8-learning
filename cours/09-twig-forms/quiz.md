# Quiz — Module 09 : Twig & Forms

Une ou plusieurs bonnes réponses : c'est indiqué. Réponses en bas.

**Q1.** (1 réponse) Quel est l'ordre des événements de formulaire lors d'une soumission (après le chargement des données) ?
- A. `PRE_SUBMIT` → `POST_SUBMIT` → `SUBMIT`
- B. `SUBMIT` → `PRE_SUBMIT` → `POST_SUBMIT`
- C. `PRE_SUBMIT` → `SUBMIT` → `POST_SUBMIT`
- D. `PRE_SET_DATA` → `SUBMIT` → `POST_SET_DATA`

**Q2.** (1 réponse) Quel événement permet de modifier les données **brutes** de la requête avant soumission ?
- A. `PRE_SET_DATA`
- B. `PRE_SUBMIT`
- C. `SUBMIT`
- D. `POST_SUBMIT`

**Q3.** (1 réponse) Dans un transformer ajouté avec `addModelTransformer()`, `reverseTransform()` convertit :
- A. model → norm
- B. norm → model (à la soumission)
- C. entité → tableau pour le Serializer
- D. requête → Response

**Q4.** (1 réponse) Quelle exception signale une erreur de transformation ?
- A. `InvalidArgumentException`
- B. `TransformationFailedException`
- C. `FormException`
- D. `UnexpectedTypeException`

**Q5.** (1 réponse) L'option `inherit_data: true` sur un sous-formulaire…
- A. rend le champ non mappé
- B. lui donne les mêmes données que le formulaire parent
- C. désactive la validation
- D. copie les erreurs du parent

**Q6.** (2 réponses) Pour qu'un `CollectionType` appelle `addTag()`/`removeTag()` et gère l'ajout/suppression d'éléments :
- A. `allow_add: true` et `allow_delete: true`
- B. `by_reference: false`
- C. `inherit_data: true`
- D. `mapped: false`

**Q7.** (1 réponse) `{% embed %}` se distingue de `{% include %}` car :
- A. il est asynchrone
- B. il permet de surcharger des blocs du template inclus
- C. il n'hérite pas du contexte
- D. il met en cache le fragment

**Q8.** (3 réponses) Une macro Twig…
- A. n'a pas accès implicitement aux variables locales du template appelant
- B. doit être importée avec `import` ou `from` si elle est dans un autre fichier
- C. peut accéder aux globals définies dans l’environnement Twig
- D. ne peut pas recevoir d'arguments

**Q9.** (1 réponse) Où le `{{ parent() }}` est-il utilisé ?
- A. Dans un bloc enfant pour afficher le contenu du même bloc du template parent
- B. Pour inclure un template
- C. Pour accéder au formulaire parent
- D. Pour appeler le contrôleur parent

**Q10.** (1 réponse) Quelle est la stratégie d'échappement par défaut pour un template `x.html.twig` ?
- A. Aucune
- B. `html`
- C. `js`
- D. `url`

**Q11.** (2 réponses) Moyens valides d'enregistrer un filtre Twig dans Symfony 8 :
- A. Méthode `getFilters()` dans une `AbstractExtension` (service autoconfiguré)
- B. Attribut `#[AsTwigFilter]` sur une méthode d'un service
- C. Attribut `#[AsFilter]` du FrameworkBundle
- D. Fichier `twig_filters.yaml` dans `config/`

**Q12.** (1 réponse) `FormEvents::POST_SUBMIT` : que faut-il vérifier avant d'utiliser les données ?
- A. `isSubmitted()` n'est jamais nécessaire
- B. `isSynchronized()`, car la transformation a pu échouer
- C. `isEmpty()`
- D. `isRoot()`

**Q13.** (1 réponse) Dans un contrôleur Symfony 8, `render('…', ['form' => $form])` avec un formulaire soumis invalide :
- A. renvoie 200
- B. renvoie 422
- C. lève une exception
- D. redirige automatiquement

**Q14.** (2 réponses) Thèmes de formulaires :
- A. Le thème global par défaut est `form_div_layout.html.twig`
- B. `{% form_theme form 'theme.html.twig' %}` applique un thème à un seul formulaire
- C. Un thème se déclare avec `framework.form.theme`
- D. Un thème ne peut pas surcharger un widget individuel

**Q15.** (1 réponse) `required: true` sur un champ :
- A. déclenche `NotBlank` côté serveur
- B. ajoute l'attribut HTML `required` seulement ; la validation serveur passe par les contraintes
- C. interdit la valeur `0`
- D. est ignoré par le navigateur

---

## Questions de mise en situation — 16 à 20

**Q16.** (1 réponse) Un POST ne contient pas le nom attendu du formulaire. Que peut faire `handleRequest()` ?

- A. Soumettre obligatoirement tout POST
- B. Laisser le formulaire non soumis
- C. Déclencher toujours un NotBlank
- D. Réparer le nom HTML automatiquement

**Q17.** (1 réponse) Un champ a subi une `TransformationFailedException`. Quelle distinction est correcte ?

- A. Une donnée valide n’a pas pu être encodée en HTML, mais le formulaire reste valide
- B. Il faut appeler le service métier avant isValid
- C. Le champ est désynchronisé ; cela diffère d’une valeur convertie mais refusée par Validator
- D. La contrainte NotBlank répare la transformation

**Q18.** (1 réponse) Un `FileType` est `mapped: false`. Quelle affirmation est vraie ?

- A. Le fichier doit être récupéré depuis le champ et ses contraintes restent applicables
- B. Le fichier échappe à la validation
- C. Le nom original est sûr pour le stockage
- D. Il est automatiquement déplacé dans public/

**Q19.** (1 réponse) Une extension de type définit `help_icon` de type string|null. Une valeur tableau produit normalement :

- A. Un message utilisateur après POST seulement
- B. Un cast automatique universel en chaîne
- C. Une erreur HTTP 403 de sécurité
- D. Une `InvalidOptionsException` à la résolution des options

**Q20.** (1 réponse) Une macro Twig lit-elle une variable locale du template appelant sans argument ?

- A. Oui, toutes les variables sont automatiquement partagées
- B. Non ; les globals Twig restent toutefois accessibles
- C. Non, et aucune globale n’est jamais disponible
- D. Uniquement si raw est utilisé

## Réponses

1. **C.** Chargement : `PRE_SET_DATA`, `POST_SET_DATA`. Soumission : `PRE_SUBMIT`, `SUBMIT`, `POST_SUBMIT` (`FormEvents.php`).
2. **B.** `PRE_SUBMIT` reçoit les données brutes de la requête.
3. **B.** Un model transformer fait model ↔ norm, un view transformer norm ↔ view. B correspond ici précisément à `addModelTransformer()` ; A est le sens de `transform()`, C et D ne sont pas des transformations de formulaire.
4. **B.** `TransformationFailedException` ⇒ champ non synchronisé et erreur de formulaire.
5. **B.** Le sous-formulaire partage les données du parent (réutilisation d'un groupe de champs sans sous-objet).
6. **A et B.** `allow_add`/`allow_delete` + `by_reference: false` (sinon les adders/removers ne sont pas appelés). Côté navigateur il faut aussi du JS pour le prototype.
7. **B.** `embed` = `include` avec blocs surchargeables.
8. **A, B et C.** La macro ne reçoit pas les variables locales de l’appelant, mais récupère les globals de l’environnement (`MacroNode`). Elle accepte des arguments : D est faux.
9. **A.** `parent()` rend le contenu du bloc du template parent.
10. **B.** Autoescape actif, stratégie `html` pour les `.html.twig`.
11. **A et B.** Extension classique autoconfigurée, ou attribut `#[AsTwigFilter]` (autoconfigure dans le twig-bundle).
12. **B.** En cas d'échec de transformation, le formulaire est désynchronisé et la donnée est celle de la requête.
13. **B.** `AbstractController::render` renvoie 422 si un formulaire invalide est passé en paramètre.
14. **A et B.** Thème global par défaut `form_div_layout.html.twig` ; `form_theme` local ; la config est `twig.form_themes` (pas `framework.form.theme`) ; les blocs (`_task_title_widget`) permettent de surcharger un widget précis.
15. **B.** `required` n'est qu'un attribut HTML ; ajoute `NotBlank` pour la validation serveur.


## Réponses raisonnées — 16 à 20

16. **B.** Méthode et nom déterminent la reconnaissance de la soumission. A néglige le nom ; C suppose la validation déjà déclenchée ; D n’a aucun mécanisme de correction automatique.
17. **C.** Conversion et validation sont deux étapes différentes. A minimise l’échec ; B utilise prématurément les données ; D prête aux contraintes un rôle de conversion.
18. **A.** Le mapping vers l’objet est désactivé, pas le traitement du champ. B confond mapping/validation ; C fait confiance au client ; D invente une publication automatique.
19. **D.** C’est une erreur de configuration du formulaire. A et C relèvent d’autres couches ; B n’est pas le comportement de setAllowedTypes().
20. **B.** Le contexte local n’est pas transmis implicitement, contrairement aux globals. A mélange les portées ; C exagère l’isolation ; D ne concerne que l’échappement.
