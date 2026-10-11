# Module 18 — Translation & localisation

[Parcours](../README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Construire une interface français/anglais, garder des clés stables, choisir la locale et diagnostiquer un message manquant. Prérequis : [Routing](../11-routing/README.md), [Twig & Forms](../09-twig-forms/README.md), [Validator](../12-validator/README.md).

**Versions :** application Symfony 8.1.8 ; ce module utilise le socle Symfony 8.0. La traduction des contenus stockés en base et les fournisseurs tiers ne font pas partie de ces exercices.

## Préparation au moment de pratiquer

`symfony/translation` n'est pas installé dans le dépôt initial. Pour commencer les exercices :

```bash
composer require symfony/translation
```

Laisse Flex appliquer sa recette, puis adapte sa configuration. `ext-intl` est présente dans l'environnement inspecté ; vérifie avec `php --ri intl` sur une autre machine, notamment pour les messages ICU en français.

## 1. Identifier un message

Un message est recherché par identifiant, domaine et locale. Des clés stables comme `catalogue.title` évitent de modifier les appels lorsque la formulation évolue. Le domaine par défaut est `messages` ; `validators` est utilisé pour les messages de contraintes.

Crée d'abord un petit dictionnaire de référence :

| Clé | Français | Anglais |
|---|---|---|
| `catalogue.title` | Catalogue | Catalogue |
| `catalogue.empty` | Aucun article | No items |
| `welcome` | Bonjour %name% | Hello %name% |

Le contenu variable est un paramètre. Concaténer le prénom dans l'identifiant créerait autant de clés que de personnes.

## 2. Utiliser le contrat de traduction

Injecte `Symfony\Contracts\Translation\TranslatorInterface`. L'appel permet d'indiquer paramètres, domaine et locale :

```php
$label = $translator->trans('welcome', ['%name%' => 'Sam'], 'messages', 'fr');
```

Cet exemple suppose le catalogue du tableau précédent. Une locale passée explicitement concerne cet appel ; elle n'est pas une demande de changer durablement toute l'application.

Les catalogues classiques peuvent être des fichiers `messages.fr.yaml` et `messages.en.yaml` sous `translations/`. Leur format et leur nom jouent chacun un rôle : un fichier valide placé dans le mauvais domaine ne résout pas une clé cherchée ailleurs.

## 3. Choisir une locale

Une route avec `_locale` rend la langue explicite dans l'URL. Le choix d'une langue préférée dans `Accept-Language` est une autre décision : l'application doit déterminer quand utiliser ce signal et comment limiter les langues supportées.

Dans le fil rouge, les seules locales d'URL admises sont `fr` et `en`. Une URL en français reste française même si le navigateur préfère l'anglais. Ce contrat rend les liens partageables et les tests déterministes.

## 4. Replis et clés manquantes

Un catalogue peut chercher dans des locales de repli. Une variante comme `fr_FR` peut utiliser `fr`, puis un repli configuré. Si aucune traduction n'est trouvée, le résultat peut être l'identifiant lui-même : une page sans exception ne prouve donc pas que ses textes sont traduits.

Dans les exercices, configure `en` comme repli. Réserve une clé uniquement anglaise pour prouver le mécanisme ; ajoute une autre clé absente partout pour détecter le cas réellement manquant. [Recherche et replis officiels](https://symfony.com/doc/current/translation.html).

## 5. Pluriels ICU

Pour les messages ICU, utilise le domaine de catalogue suffixé `+intl-icu`, par exemple `messages+intl-icu.fr.yaml`. L'appel conserve le domaine logique `messages`. Les paramètres ICU utilisent des noms comme `count`, sans les `%` des substitutions classiques.

Exemple réduit, à adapter dans les exercices :

```yaml
basket.count: '{count, plural, =0 {Panier vide} one {# article} other {# articles}}'
```

La branche exacte `=0` exprime une intention explicite ; les catégories linguistiques ne se résument pas à `count === 1` dans toutes les langues. Teste au minimum 0, 1 et 2, et n'assemble pas un pluriel par concaténation dans Twig.

## 6. Twig, formulaires et validation

Dans Twig, `{{ 'catalogue.title'|trans }}` utilise le traducteur et reste soumis à l'échappement de la sortie. Évite de déclarer un message entier comme HTML sûr uniquement pour afficher un paramètre utilisateur.

Un formulaire peut choisir son domaine de traduction pour ses labels. Les erreurs de contraintes se règlent dans le domaine de validation. Une page peut donc avoir des labels français et des erreurs anglaises si un seul domaine a été traduit.

`TranslatableMessage` permet de transporter un identifiant et ses paramètres jusqu'au moment du rendu. Il ne contient pas automatiquement la traduction définitive : la locale au moment de sa résolution compte.

## Observer et vérifier

Après installation et création des premiers catalogues :

```bash
php bin/console debug:translation fr
php bin/console debug:translation en
php bin/console debug:config framework translator
php bin/console lint:yaml translations/
php bin/console help translation:extract
php bin/phpunit tests/Translation
```

Une extraction peut manquer une clé construite dynamiquement. Un lint YAML valide la syntaxe YAML ; il ne suffit pas à prouver que tous les messages ICU se formatent correctement avec les paramètres attendus. Ajoute des tests d'appels réels au traducteur et un parcours HTTP par langue.

## Sources

- [Traductions](https://symfony.com/doc/current/translation.html), [format ICU](https://symfony.com/doc/current/translation/message_format.html) — sélectionner la version étudiée.
- [Translator Symfony 8.0](https://github.com/symfony/translation/blob/v8.0.0/Translator.php) : résolution du domaine ICU et des replis.
- Après installation : `vendor/symfony/translation/Translator.php` et les catalogues générés par les exercices.
