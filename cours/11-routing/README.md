# Module 11 — Routing : reconnaître et générer les URL

[Parcours](../README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Savoir prédire quelle route gagne, expliquer un 404/405 et générer une URL correcte dans plusieurs langues et domaines.
Prérequis : [HTTP](../04-http-rfc9110/README.md), contrôleurs et [conteneur](../03-dependency-injection/README.md).
Le fil rouge est un catalogue public de trois livres conservés dans un tableau ; aucune persistence ni authentification.
Code futur : `src/Routing/`, tests : `tests/Routing/`. Les exemples de cette fiche sont indépendants des exercices.
Le dépôt exécute Symfony **8.1.8**, PHP **8.4.1** ; les notions évaluées ici appartiennent au socle **8.0**.

## 1. Deux directions, deux contrats

Le matcher transforme une requête en paramètres : nom de route, contrôleur, variables du chemin, valeurs par défaut.
Le générateur transforme un **nom de route** et des paramètres en URL : il ne recherche pas un contrôleur par son nom PHP.
Une route connue peut générer une URL même si le livre demandé n'existe pas dans le catalogue.
L'existence métier relève du service puis du contrôleur, pas du matcher.
La query string n'entre pas dans la comparaison du chemin : `/books?page=2` et `/books?page=3` utilisent la même route.

```php
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

final class ArchiveController extends AbstractController
{
    #[Route('/archives/{year}', name: 'archive_year',
        requirements: ['year' => '\\d{4}'], methods: ['GET'])]
    public function year(int $year): Response
    {
        return new Response((string) $year);
    }
}
```

`/archives/2025` est candidat ; `/archives/hier` ne l'est pas. Le type `int` n'est pas un remplacement du requirement.
Une mauvaise méthode sur un chemin reconnu conduit normalement à **405**, avec les méthodes autorisées.
Une absence de route correspondante conduit à **404** ; un 404 métier peut aussi arriver après le routage.

## 2. Contraintes, défauts et ordre

Un placeholder accepte normalement un segment, donc pas `/` ; un requirement explicite peut changer ce comportement.
Les requirements sont des expressions régulières sans délimiteurs PHP : `\d+`, pas `/\d+/`.
`Requirement::DIGITS` dans `Symfony\Component\Routing\Requirement` évite certaines expressions répétées.
Une valeur par défaut rend un paramètre optionnel si la structure du chemin le permet ; les segments suivants comptent.
Pour `/archives/{year}/{page}`, rendre seulement `year` optionnel ne permet pas de deviner `page` à sa place.
Les valeurs par défaut ne sont pas validées comme les paramètres fournis : choisir des défauts cohérents reste ta responsabilité.

```php
#[Route('/archives/{page}', name: 'archive_index',
    requirements: ['page' => '\\d+'], defaults: ['page' => 1], methods: ['GET'])]
// Attribut à placer sur une action recevant int $page.
```

En présence de candidats concurrents, l'ordre de la collection compte ; ne suppose pas un classement « plus spécifique d'abord ».
Avec les attributs, une `priority` plus haute place une route avant celles de priorité inférieure.
Exemple de collision : `/books/{slug}` peut absorber `/books/search` si `search` respecte le requirement de `slug`.
Une priorité explicite et/ou un requirement adapté rend l'intention lisible ; un test protège le mot réservé.
Les noms doivent être uniques ; une collision de noms peut remplacer une entrée au chargement.

## 3. Paramètres internes et attributs de requête

| Paramètre | Rôle |
|---|---|
| `_controller` | cible de l'exécution, résolue après le matching |
| `_route` | nom de la route reconnue |
| `_route_params` | paramètres associés à cette reconnaissance |
| `_locale` | locale de la requête, par exemple `fr` |
| `_format` | format de requête ; ce n'est pas une sérialisation automatique |
| `_fragment` | fragment fourni au générateur ; jamais transmis par le navigateur au serveur |

Dans une action, `Request::attributes->get('_route')` observe le nom reconnu.
Les paramètres de route se trouvent dans les attributs, les paramètres `?page=2` dans `Request::query`.
Une valeur par défaut supplémentaire peut transporter un choix interne ; elle ne devient pas automatiquement un segment.
N'utilise pas un paramètre de route comme preuve d'autorisation : routage et contrôle d'accès ont des responsabilités distinctes.

## 4. Générer sans concaténer

```php
use Symfony\Component\Routing\Generator\UrlGeneratorInterface;

// $urls est injecté par le constructeur du service.
$path = $urls->generate('archive_year', ['year' => 2025]);
$url = $urls->generate('archive_year', ['year' => 2025],
    UrlGeneratorInterface::ABSOLUTE_URL);
```

`ABSOLUTE_PATH` est le mode par défaut ; `ABSOLUTE_URL` inclut schéma et hôte.
`RELATIVE_PATH` et `NETWORK_PATH` répondent à d'autres usages ; ce dernier commence par `//hote`.
Un paramètre fourni en trop est généralement ajouté à la query string ; `_fragment` donne `#...`.
Pour forcer des paramètres de query, `_query` accepte un **tableau** ; une autre valeur lève une exception en 8.0.
Un paramètre obligatoire absent produit `MissingMandatoryParametersException`.
Un nom inconnu produit `RouteNotFoundException` ; une valeur incompatible peut produire `InvalidParameterException`.
Le contexte fournit l'hôte et le schéma : en CLI, il ne provient pas d'une requête navigateur.
Configurer `framework.router.default_uri` est adapté aux liens de commandes ; observer d'abord `debug:config framework router`.
En Twig : `path('archive_year', {year: 2025})` ou `url(...)`, jamais une concaténation manuelle.

## 5. Hôtes, schémas et locales

```php
#[Route(path: ['fr' => '/archives/{year}', 'en' => '/archive/{year}'],
    name: 'localized_archive', requirements: ['year' => '\\d{4}'],
    host: 'library.example.test', methods: ['GET'])]
// Attribut d'action ; aucune configuration DNS n'est nécessaire avec KernelBrowser.
```

Une contrainte `host` fait partie de la reconnaissance : le bon chemin sur un mauvais hôte ne suffit pas.
Un hôte peut contenir des placeholders, eux aussi contraints et utilisés par la génération.
Une route localisée produit des variantes ; demander le nom logique avec `_locale => 'en'` choisit l'URL anglaise.
Cela ne traduit ni le texte métier ni un slug : les traductions du contenu relèvent d'une autre couche.
`_locale` est aussi utilisable comme segment `/{_locale}/books`, avec requirement `fr|en`.
Une locale par défaut n'est pas une liste de locales autorisées : le requirement doit exprimer cette liste.
`schemes: ['https']` influence matching et génération ; les redirections du routeur HTTP méritent un test dédié.
Dans `WebTestCase`, passer `HTTP_HOST` et, si nécessaire, `HTTPS` pour reproduire le contexte voulu.

## 6. Observer avant de corriger

```bash
symfony console debug:router
symfony console debug:router archive_year
symfony console router:match /archives/2025 --method=GET
symfony console router:match /archives/2025 --method=POST
symfony console router:match /archives/2025 --host=library.example.test
symfony console debug:config framework router
php bin/phpunit tests/Routing
```

Remplacer `symfony console` par `php bin/console` si la CLI Symfony manque.
Le projet charge les attributs des contrôleurs services via la recette ; aucune route YAML à ajouter.
Devant un 404 : vérifier environnement, route enregistrée, ordre, host, méthode, requirements, puis service métier.
Lire `var/log/dev.log` pour distinguer absence de route et exception du contrôleur.
Le profiler est absent du dépôt initial : son usage est facultatif, après `composer require --dev symfony/profiler-pack`.
Un test HTTP observe statut **et contenu**, un test du générateur observe chemin **et contexte**.

## 7. Pièges et repères de version

- `priority` règle un ordre de reconnaissance ; ce n'est ni une permission ni une préférence du générateur.
- `_format: json` ne remplace pas `JsonResponse` ni le Serializer.
- Une route GET peut aussi servir HEAD ; tester les sémantiques HTTP, pas seulement le nom de méthode PHP.
- Une redirection vers une URL générée ne garantit pas l'existence de la ressource métier.
- Depuis **8.0**, utiliser `Routing\Attribute\Route` ; les alias `Annotation` et le format XML ont été retirés.
- La ressource `routing.controllers` vient de **7.4**, elle n'est pas une nouveauté 8.1.
- **Hors certification 8.0 :** 8.1 ajoute une option PHP `trailingSlashOnRoot` sur `CollectionConfigurator::prefix()` ; les ateliers restent en attributs.

## Sources officielles et suite

Les règles de matching, génération, hôtes et locales sont détaillées dans [Routing 8.0](https://symfony.com/doc/8.0/routing.html).
Les [bonnes pratiques Symfony](https://symfony.com/doc/8.0/best_practices.html) expliquent le choix des attributs.
Vérification locale : `vendor/symfony/routing/CHANGELOG.md`, `Attribute/Route.php`, `Generator/UrlGenerator.php`.
Poursuivre avec [Validator](../12-validator/README.md) et revenir aux cas HTTP du [module 04](../04-http-rfc9110/README.md).
