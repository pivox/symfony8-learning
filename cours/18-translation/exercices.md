# Exercices — Module 18

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md) · [Méthode](../README.md)

Installer Translation selon le cours avant de commencer. Code dans `src/Translation/`, tests dans `tests/Translation/`, catalogues dans `translations/`, templates dans `templates/translation/`. Utilise `KernelTestCase` pour le traducteur intégré et `WebTestCase` pour les pages.

## 1. Guidé — deux catalogues

Crée `catalogue.title` et `catalogue.empty` dans les catalogues français et anglais avec les textes du cours.

- **Critères :** appels explicites au traducteur retournant `Catalogue`, `Aucun article` et `No items` selon clé et langue ; aucune clé brute dans les résultats attendus.
- **Indice :** observe séparément l'identifiant, le domaine et la locale.

## 2. Guidé — interpolation

Ajoute `welcome` puis crée un service `Greeting` qui reçoit un nom et une locale.

- **Critères :** `Sam/fr` donne `Bonjour Sam`, `Sam/en` donne `Hello Sam` ; le nombre de clés ne varie pas quand le nom change. Un nom vide est rejeté par le service avant traduction.
- **Indice :** la valeur utilisateur appartient aux paramètres, pas à l'identifiant.

## 3. Autonome — locale dans l'URL

Crée `/training/{_locale}/catalogue`, limitée à `fr|en`, avec une route PHP par attribut et un contrôleur fin rendant Twig.

- **Critères :** français pour `/fr/` même avec `Accept-Language: en` ; anglais pour `/en/` ; locale `de` non admise et 404 ; liens générés avec `path()` conservant la langue.
- **Indice :** la politique du cours privilégie la locale explicite de l'URL.

## 4. Diagnostic — repli ou oubli

Configure `en` comme repli. Ajoute `catalogue.help` uniquement en anglais avec `Ask for help`, et n'ajoute jamais `catalogue.unknown`.

- **Critères :** en français, `catalogue.help` donne le texte anglais ; `catalogue.unknown` reste une clé ; le test distingue ces deux situations. Relève le diagnostic de `debug:translation fr`.
- **Indice :** un fallback trouvé et un identifiant jamais traduit ne sont pas le même état.

## 5. Autonome — pluriels ICU

Définis `basket.count` en français et anglais. Contrat : 0 → `Panier vide` / `Empty basket`, 1 → `1 article` / `1 item`, 2 → `2 articles` / `2 items`.

- **Critères :** six combinaisons testées via le traducteur réel ; le template ne choisit pas lui-même le singulier ; `ext-intl` disponible ; message situé dans le bon catalogue ICU.
- **Indice :** distingue paramètre ICU `count` et substitution classique `%count%`.

## 6. Autonome — formulaire bilingue

Crée un formulaire de contact sur un DTO sans persistance, avec nom obligatoire. Utilise un domaine `forms` pour le label, `validators` pour une contrainte dont la clé est `contact.name.required`.

- **Critères :** label `Nom` / `Name`, erreur `Indique ton nom` / `Enter your name` après soumission vide ; CSRF conservé et testé avec un formulaire réellement chargé. Aucune donnée n'est envoyée par email.
- **Indice :** le domaine des labels n'est pas automatiquement celui des contraintes.

## 7. Diagnostic — échappement et clé dynamique

Rends le message d'accueil avec le nom `<script>alert(1)</script>`. Compare ensuite l'extraction d'une clé littérale et d'une clé assemblée dynamiquement.

- **Critères :** aucun élément `script` issu du nom dans le DOM ; texte affiché comme donnée ; note les limites de l'extracteur et complète les tests pour les clés dynamiques nécessaires.
- **Indice :** ne résous pas une erreur de traduction en ajoutant `raw`.

## 8. Défi final — page de catalogue complète

Assemble titre, état vide, compteur ICU, lien de changement de langue et formulaire. Tous les articles sont des fixtures en mémoire.

- **Critères :** parcours HTTP français puis anglais ; compteur pour 0 et 2 articles ; formulaire invalide traduit ; pas de fuite de langue entre requêtes et aucune clé brute parmi les messages requis.
- **Livrable :** tests et tableau des domaines utilisés avec leur responsabilité.
- **Indice :** teste aussi une langue après l'autre avec le client de test pour détecter les états conservés involontairement.

## 9. Diagnostic — un texte identique à sa clé peut être traduit

**Niveau :** intermédiaire · **Durée indicative :** 45 minutes.
**Prérequis :** Translation installé via Composer/Flex, exercices 1 et 4, [inspection des catalogues](approfondissement.md).

Scénario : un contrôle de qualité doit distinguer une traduction française rédigée, un repli anglais et une clé absente.
Dans le domaine `audit`, définis `title: Contrôle` en français et `title: Check` en anglais ; définis `OK: OK` dans les deux langues.
Définis `help: Ask for help` uniquement en anglais ; laisse `missing` absent partout et conserve `en` comme repli.
Crée un service d'audit recevant les quatre identifiants dans l'ordre `title`, `OK`, `help`, `missing`.

- **Critères :** un appel réel en français retourne les états `local`, `local`, `fallback`, `missing` dans cet ordre ; les textes associés sont `Contrôle`, `OK`, `Ask for help`, `missing`.
- **Critères :** l'audit repose sur le catalogue réel du traducteur configuré ; `debug:translation fr --domain=audit` fournit un diagnostic cohérent.
- **Cas négatif :** un algorithme qui compare seulement le texte traduit à l'identifiant classe à tort `OK` comme absent ; le test doit détecter cette confusion.
- **Livrable :** `KernelTestCase` vérifiant états et textes ; domaine explicite dans chaque appel ; aucune modification de la locale globale nécessaire pour l'audit.
- **Indice 1 :** la disponibilité par repli et la présence dans le catalogue demandé répondent à deux questions.
- **Indice 2 :** compare `MessageCatalogueInterface::defines()` et `has()` ; pour l'inspection, vise `TranslatorBagInterface` et vérifie son câblage vers le service `translator`.

## 10. Autonome — résoudre un message et son paramètre dans la même langue

**Niveau :** intermédiaire · **Durée indicative :** 45 minutes.
**Prérequis :** Translation installé, exercice 2 et [messages différés](approfondissement.md).

Scénario : un service prépare une notification d'état avant de savoir si elle sera rendue en français ou en anglais.
Dans le domaine `notifications`, définis `delivery.summary` par `Livraison %reference% : %state%` et `Delivery %reference%: %state%`.
Définis `delivery.ready` par `prête` et `ready` dans ces mêmes catalogues.
Prépare un `TranslatableMessage` extérieur pour la référence `R-42`, avec un autre message traduisible comme paramètre d'état.

- **Critères :** le même message préparé, résolu successivement en `fr`, `en`, puis `fr`, donne `Livraison R-42 : prête`, `Delivery R-42: ready`, puis le premier texte identique.
- **Critères :** l'objet préparé conserve l'identifiant et les paramètres ; aucun appel prématuré au traducteur n'est nécessaire lors de sa construction.
- **Cas négatif :** avec une clé d'état volontairement absente `delivery.unknown`, le rendu révèle cette clé brute ; le test ne considère pas ce résultat comme une notification traduite valide.
- **Livrable :** tests du service et appels au traducteur intégré avec locales explicites ; démontrer qu'un état traduit d'avance en français échouerait sur le cas anglais.
- **Indice 1 :** reporte la traduction des valeurs qui sont elles-mêmes des messages ; la référence `R-42` reste une donnée ordinaire.
- **Indice 2 :** `TranslatableMessage::trans()` résout les paramètres implémentant `TranslatableInterface` avec la locale transmise avant de traduire le message extérieur.
