# Approfondissement — auditer les catalogues et traduire au bon moment

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Une page peut afficher un texte lisible tout en cachant une traduction manquante dans sa langue.
Un message peut aussi être traduit trop tôt, avant que la locale de son destinataire soit connue.
Ce complément approfondit ces deux problèmes avec des API du socle Symfony 8.0.
Le dépôt tourne en 8.1.8, mais `symfony/translation` est absent au départ.
Au moment de pratiquer : `composer require symfony/translation`, puis laisser Flex appliquer sa recette.
Les catalogues des laboratoires restent dans `translations/` et n'utilisent aucune base de données.

## 1. Trouvé ici, trouvé ailleurs ou absent partout ?

Le contrat usuel `Symfony\Contracts\Translation\TranslatorInterface` expose la traduction d'un message.
L'inspection des catalogues relève d'un autre contrat : `Symfony\Component\Translation\TranslatorBagInterface`.
Sa méthode `getCatalogue('fr')` retourne le catalogue et donne accès à sa chaîne de repli.
Ne suppose pas que tous les traducteurs tiers exposent cet accès en plus du contrat de traduction.
Dans l'application, inspecte le service `translator` avant de choisir son injection pour un outil de diagnostic.
Un `#[Autowire(service: 'translator')]` peut expliciter le service si aucun alias du type d'inspection n'est disponible.

Le catalogue distingue deux questions :

```php
// $catalogue est un MessageCatalogueInterface obtenu pour une locale donnée.
$local = $catalogue->defines('notice.title', 'messages');
$available = $catalogue->has('notice.title', 'messages');
```

`defines()` vérifie ce catalogue ; `has()` peut également trouver la clé dans les catalogues de repli.
Une traduction disponible n'est donc pas nécessairement une traduction rédigée pour la langue demandée.
Un audit peut classer les clés en locales, héritées ou manquantes au lieu d'afficher un simple booléen.
Une clé héritée peut être acceptable temporairement, mais elle doit rester visible dans le diagnostic.
Le [code du catalogue 8.0.0](https://github.com/symfony/translation/blob/v8.0.0/MessageCatalogue.php) expose cette distinction.

Ne détecte pas une absence avec la seule comparaison `trans($id) === $id`.
Une vraie traduction de l'identifiant `OK` peut être précisément `OK`.
L'identité du texte ne suffit alors pas à distinguer traduction présente et absence de traduction.
De même, deux locales peuvent légitimement partager le mot `Catalogue`.
Teste les métadonnées du catalogue pour sa couverture, puis le texte rendu pour le contrat visible.

| `defines()` | `has()` | Interprétation |
|---|---|---|
| Vrai | Vrai | Clé définie dans le catalogue demandé |
| Faux | Vrai | Clé obtenue par repli |
| Faux | Faux | Clé absente de toute la chaîne inspectée |
| Vrai | Faux | Incohérent pour le catalogue Symfony standard |

## 2. Reporter la traduction jusqu'au rendu

`TranslatableMessage` transporte l'identifiant, les paramètres et le domaine d'une traduction future.
Il ne fixe pas une locale dans son constructeur.
L'appel `trans($translator, $locale)` choisit ensuite la langue de résolution.
Cela permet à un service de produire une intention de message sans dépendre de la langue courante de la requête.
Une opération différée doit néanmoins transporter la locale souhaitée si elle dépend d'un destinataire.
Le worker ne peut pas deviner la langue d'une requête qui n'existe plus.

```php
use Symfony\Component\Translation\TranslatableMessage;

$message = new TranslatableMessage('notice.ready', ['%reference%' => 'R-7'], 'notices');
// Plus tard, avec un traducteur réel et les catalogues requis :
$text = $message->trans($translator, 'fr');
```

La création de l'objet ne prouve donc rien sur l'existence de `notice.ready` dans un catalogue.
Le test doit réellement résoudre le message et vérifier le texte dans chaque langue attendue.
Un paramètre peut lui-même implémenter `TranslatableInterface`.
`TranslatableMessage` résout ces paramètres traduisibles avec la locale reçue, avant le message extérieur.
Cela évite de figer prématurément un nom d'état en français au milieu d'une phrase anglaise.
Le [code de TranslatableMessage 8.0.0](https://github.com/symfony/translation/blob/v8.0.0/TranslatableMessage.php) montre cette résolution.

Un paramètre ordinaire reste une donnée : il n'est pas traduit simplement parce que c'est une chaîne.
Le nom d'une personne, une référence ou un identifiant ne deviennent pas des clés de catalogue implicitement.
Au rendu HTML, le résultat reste soumis à l'échappement approprié ; la traduction ne garantit pas du HTML sûr.
Un message structuré peut être réutilisé dans une réponse texte ou un template, avec des tests distincts.

## 3. Couverture et comportement sont deux vérifications

Un audit de catalogue répond à « avons-nous rédigé cette clé dans cette langue ? ».
Un test de rendu répond à « ce lecteur voit-il le texte attendu avec ces paramètres ? ».
Une clé présente mais contenant `%reference%` mal orthographié passe le premier contrôle et échoue au second.
Un repli anglais peut passer un test de texte non vide tout en manquant la couverture française attendue.
Définis une liste déterministe de clés obligatoires, domaines et locales pour l'atelier.
N'utilise pas une extraction dynamique comme seule source de vérité pour cette liste.
Une clé fabriquée dans du code peut échapper à l'extracteur ; le contrat de ton interface reste requis.
La [documentation Translation 8.0](https://symfony.com/doc/8.0/translation.html) présente extraction et diagnostic.

## Observer après installation

```bash
php bin/console debug:container translator
php bin/console debug:autowiring Translator
php bin/console debug:translation fr
php bin/console debug:translation en
php bin/console lint:yaml translations/
php bin/phpunit tests/Translation
```

Compare l'audit programmatique et `debug:translation` sur les mêmes clés.
Note le domaine explicitement : une clé dans `notices` ne prouve pas sa présence dans `messages`.
Avant installation, les classes du composant ne sont pas disponibles simplement parce que les Contracts existent.
Le [contrat d'inspection 8.0.0](https://github.com/symfony/translation/blob/v8.0.0/TranslatorBagInterface.php) définit cet accès aux catalogues.
Après installation, relis `MessageCatalogue.php` et `TranslatableMessage.php` sous `vendor/symfony/translation/`.
Les [exercices 9 et 10](exercices.md) demandent un audit précis puis une résolution différée bilingue.
