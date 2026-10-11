# Module 01 — PHP 8.4 & PSR

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

> **Périmètre** : application vérifiée en **Symfony 8.1.8**, PHP ≥ 8.4 ; la [certification Symfony 8](https://certification.symfony.com/exams/symfony.html) vise **Symfony 8.0**. Les nouveautés marquées **8.1** sont un complément de pratique, pas des exigences de cet examen. Suivre la [méthode de travail du parcours](../README.md) : prédire, observer, tester, expliquer.

## Objectifs et prérequis

Prérequis : classes, interfaces, tableaux et premiers tests PHPUnit. À la fin, tu dois expliquer les invariants d'un objet PHP, distinguer interface et implémentation, et prédire les erreurs de typage sans exécuter le code. Les hooks ne remplacent pas les fondamentaux : complète aussi les critères des exercices déjà commencés avant de passer aux nouveautés.

Symfony 8 exige **PHP ≥ 8.4**. Ce module couvre les nouveautés du langage que tu croiseras dans le code Symfony 8, puis les PSR utilisées par le framework.

## 1. Property hooks (8.4)

Un hook `get`/`set` s'attache directement à une propriété : plus besoin de getter/setter triviaux.

```php
class User
{
    public string $fullName {
        get => $this->first . ' ' . $this->last;   // propriété virtuelle (pas de stockage)
    }

    public string $email {
        set (string $value) {                       // hook avec corps
            $this->email = strtolower($value);      // $this->email dans son propre hook = valeur stockée
        }
    }

    public function __construct(private string $first, private string $last, string $email)
    {
        $this->email = $email;
    }
}
```

À retenir :
- Une propriété **sans** stockage (aucun hook ne lit/écrit `$this->prop`) est *virtuelle*.
- Forme courte : `set => expr;` ; le hook `set` peut prendre un type plus large que la propriété.
- Les hooks sont autorisés dans les interfaces (`public string $name { get; }`) : l'interface impose une *lecture*, pas un stockage.
- Compatibles avec la promotion de constructeur.

## 2. Asymmetric visibility (8.4)

```php
class Money
{
    public function __construct(
        public private(set) int $amount,   // lisible partout, modifiable seulement dans la classe
    ) {}
}
```

- La visibilité du `set` doit être **égale ou plus restrictive** que celle du `get`.
- Requiert une propriété **typée**.
- Alternative légère à `readonly` quand la classe doit pouvoir se modifier elle-même.

## 3. `new` sans parenthèses (8.4)

```php
$name = new Foo()->bar();   // avant 8.4 : (new Foo())->bar()
```

## 4. Lazy objects (8.4)

`ReflectionClass::newLazyGhost()` / `newLazyProxy()` : l'objet n'est initialisé qu'au premier accès à son état.
**Lien Symfony** : Symfony peut utiliser ces objets natifs pour les *lazy services* (`#[Lazy]`) — voir module 03.

## 5. Fonctions de tableau 8.4

`array_find`, `array_find_key`, `array_any`, `array_all` — plus besoin de `array_filter` + `reset`.

## 6. PSR utilisées par Symfony

| PSR | Rôle | Où dans Symfony |
|-----|------|-----------------|
| 3 | Logger | `Psr\Log\LoggerInterface` (implémentation à observer ; Monolog optionnel) |
| 4 | Autoloading | `composer.json` → `autoload.psr-4` |
| 6 / 16 | Cache / Simple cache | Cache component (`CacheInterface` de Symfony Contracts est l'API recommandée) |
| 7 / 17 / 18 | HTTP message / factory / client | `symfony/psr-http-message-bridge`, HttpClient |
| 11 | Container | `ContainerInterface` (`get`, `has`) |
| 14 | Event dispatcher | `EventDispatcherInterface` |
| 15 | HTTP handlers/middleware | adaptation nécessaire ; le bridge PSR-7 convertit les messages, pas le contrat de handler |
| 20 | Clock | `Symfony\Component\Clock\ClockInterface` |

## Piège certification
- `private(set)` vs `readonly` : `readonly` interdit toute réécriture hors initialisation, `private(set)` l'autorise depuis la classe.
- Un hook `get` seul sur une propriété sans stockage rend la propriété **en lecture seule**.
- PSR-11 n'a pas de `set` : l'API du container Symfony n'est pas dans la PSR.

## Observer dans l'app
- `php -r 'echo PHP_VERSION;'` et `composer.json` (`"php": ">=8.4"`).
- `bin/console debug:container --types | rg -i psr` : quelles interfaces PSR sont câblées.
- `bin/console debug:autowiring Psr` : alias PSR disponibles à l'injection.
- `bin/console debug:autowiring clock`.

## Doc officielle
- https://www.php.net/releases/8.4/fr.php
- https://www.php-fig.org/psr/


## 7. Typage, identité et contrat métier

Un type PHP définit ce qui peut entrer ; une règle métier définit ce qui est acceptable. `int $quantity` exclut un objet mais autorise `-3` : cette dernière valeur demande une validation. `?T` signifie `T|null`, une union `A|B` accepte l'un ou l'autre, une intersection `A&B` exige les deux interfaces. La signature d'une implémentation peut élargir ses paramètres (contravariance) et préciser son retour (covariance), pas l'inverse.

```php
declare(strict_types=1);

function twice(int $n): int { return $n * 2; }
twice('4'); // TypeError dans cet appel depuis un fichier strict
```

Pour les paramètres scalaires, c'est le fichier **appelant** qui décide du mode strict ; ce n'est pas un interrupteur global du projet. `strict_types` ne valide ni une adresse email ni une forme de tableau. `readonly` empêche la réaffectation d'une propriété après initialisation, pas la mutation interne d'un objet qu'elle contient : préférer `DateTimeImmutable` pour une date de value object.

## 8. Enums, exceptions, closures et traits

Une enum représente un ensemble fermé ; sa valeur scalaire est un format d'échange, pas un cas d'enum.

```php
enum PublicationState: string
{
    case Draft = 'draft';
    case Published = 'published';
}

PublicationState::from('draft') === PublicationState::Draft; // true
PublicationState::tryFrom('missing'); // null
// from('missing') lève ValueError ; cases() retourne les cas.
```

Utiliser `match` rend visibles les branches du domaine : sans branche applicable ni `default`, PHP lève `UnhandledMatchError`. Une exception métier exprime une opération impossible ; elle n'est pas un code HTTP. Les deux familles `Exception` et `Error` implémentent `Throwable` ; `catch (Exception)` ne capture donc pas une `TypeError`. `finally` s'exécute aussi quand un bloc retourne ou lance une exception : pratique pour libérer une ressource, mais un `return` dans `finally` peut masquer le résultat initial.

```php
$rate = 2;
$byValue = fn (int $n): int => $n * $rate;
$byReference = function (int $n) use (&$rate): int { return $n * $rate; };
$rate = 3;
// $byValue(10) vaut 20 ; $byReference(10) vaut 30.
```

Une closure permet de différer un calcul ou d'injecter un comportement. Une arrow function capture implicitement par valeur ; `use (&$x)` partage la variable. Cela explique pourquoi un compteur de test doit être observé au bon endroit.

Un trait réutilise du code ; il ne constitue pas un type injectable. Deux traits fournissant la même méthode créent un conflit à résoudre avec `insteadof` ; `as` crée un alias ou adapte la visibilité, mais ne résout pas seul le conflit. Préférer une interface pour le contrat et la composition lorsqu'un comportement doit être remplaçable.

### Vérifier les PSR sans supposer un adaptateur

`Psr\Log\LoggerInterface` définit le contrat PSR-3 ; Monolog n'est qu'une implémentation possible et n'est pas installé ici. `Symfony\Contracts\EventDispatcher\EventDispatcherInterface` et `Psr\EventDispatcher\EventDispatcherInterface` sont des noms distincts : vérifier l'alias avant de prétendre pouvoir injecter l'un ou l'autre. Un bridge PSR-7 convertit des messages ; cela ne transforme pas automatiquement HttpKernel en handler PSR-15.

Observer avec `bin/console debug:autowiring Psr`, lire `vendor/psr/`, puis exécuter les petits exemples dans un fichier PHP isolé. Pour une erreur volontaire de compilation (conflit de traits), utiliser un fichier séparé afin de ne pas empêcher toute la suite PHPUnit de démarrer. Sources : [enums PHP](https://www.php.net/manual/en/language.enumerations.backed.php), [objets paresseux](https://www.php.net/manual/en/language.oop5.lazy-objects.php).
