# Module 14 — Tests et diagnostic : prouver un comportement

[Parcours](../README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Choisir le niveau de test utile, isoler les dépendances et transformer un incident en test de régression.
Lire un échec jusqu'à sa cause sans désactiver l'assertion qui le révèle.
Prérequis : [kernel](../02-kernel-architecture/README.md), [DI](../03-dependency-injection/README.md), [HTTP](../04-http-rfc9110/README.md), modules 11–13.
Fil rouge : mini-catalogue public en mémoire, recherche HTTP et rapport Console ; aucune base ni authentification.
Code futur : `src/Testing/`, tests : `tests/Testing/`. Réutiliser les modules précédents seulement après les avoir réalisés.
Le dépôt contient PHPUnit **13.3.6**, BrowserKit et CssSelector ; `php bin/phpunit` est disponible.
Symfony **8.1.8**, PHP **8.4.1** ; les API Symfony principales de cette fiche appartiennent au socle **8.0**.

## 1. Une assertion doit porter sur le contrat

Un test arrange une situation, appelle le système comme le ferait son utilisateur et vérifie un résultat observable.
« Aucune exception » ne prouve ni un tri correct, ni un statut HTTP, ni l'absence d'un effet de bord.
Le nom du test décrit une règle : `testUnknownBookReturns404`, pas seulement `testShow`.
Commencer par un exemple nominal puis une frontière : absent, vide, invalide, duplication, dépassement.
Une régression est un cas minimal qui échoue avec le défaut et réussit avec la correction.
La couverture de lignes aide à localiser une zone oubliée ; elle ne mesure pas la qualité des assertions.

| Objet étudié | Base de test du parcours | Frontière exercée |
|---|---|---|
| DTO, valeur ou fonction pure | `PHPUnit\Framework\TestCase` | appel PHP sans kernel |
| Service de l'application | `KernelTestCase` | service réel résolu par le conteneur |
| Contrôleur et parcours HTTP | `WebTestCase` | requête, routage, listeners, réponse |
| Commande de l'application | `KernelTestCase` + tester Console | enregistrement, entrées, sortie et statut |

## 2. Tests d'objets purs et jeux de données

Instancier directement un objet valeur évite de dépendre inutilement des bundles et de l'environnement.
Comparer une valeur attendue avec `assertSame()` détecte aussi une erreur de type que `assertEquals()` peut laisser passer.
Un provider de données permet de nommer les frontières d'un même contrat, sans dupliquer le corps du test.
Avec la version PHPUnit du dépôt, utiliser les attributs et des providers `public static`.

```php
use PHPUnit\Framework\Attributes\DataProvider;
use PHPUnit\Framework\TestCase;

final class TrimExampleTest extends TestCase
{
    #[DataProvider('inputs')]
    public function testWhitespace(string $input, string $expected): void
    {
        self::assertSame($expected, trim($input));
    }

    public static function inputs(): iterable
    {
        yield 'vide' => ['', ''];
        yield 'bords' => [' Ada ', 'Ada'];
    }
}
```

Cet exemple explique le mécanisme ; les exercices doivent tester tes objets et leurs règles, pas réécrire les tests de PHP.

## 3. Services et conteneur de test

`KernelTestCase::bootKernel()` démarre un kernel de test ; `self::getContainer()` expose le conteneur prévu pour les tests.
Il permet d'accéder aux services privés conservés, mais ne recrée pas ceux supprimés/inlinés par compilation.
La configuration `framework.test` et `KERNEL_CLASS` doivent être présentes : vérifier la recette et `.env.test` avant d'accuser le test.
Ne pas modifier `.env` avec des secrets pour réparer un démarrage ; les variables réelles restent locales.
Un service privé introuvable peut être inutilisé et supprimé ; comprendre sa consommation avant de le rendre public.
Le test appelle une méthode utile et vérifie son résultat, pas seulement `assertInstanceOf()` sur le conteneur.

```php
// Dans une méthode d'un KernelTestCase, après création du service de l'atelier :
self::bootKernel();
$catalogue = self::getContainer()->get(\App\Testing\Catalogue::class);
$result = $catalogue->search('symfony');
self::assertCount(2, $result);
```

Un tel test suppose une fixture définie explicitement ; le nombre `2` n'est pas une propriété universelle du catalogue.

## 4. Doubles : maîtriser une dépendance

Un stub fournit une réponse contrôlée ; un mock ajoute des attentes sur des appels pertinents pour le contrat.
Doubler une interface stable, pas tous les objets de la chaîne ; un surcroît d'attentes rend les refactorings inutilement coûteux.
Un faux catalogue en mémoire peut être plus lisible qu'une longue succession de `willReturn()`.
Pour un service résolu par DI, remplacer la dépendance avec `self::getContainer()->set(Interface::class, $double)` **avant** de récupérer le consommateur.
Le remplacement utilise l'identifiant réellement injecté ; un alias et sa cible méritent une vérification par `debug:container`.
Un service déjà initialisé ne peut généralement plus être remplacé ; redémarrer proprement le kernel plutôt que modifier ses propriétés internes.
Pour les délais et dates, privilégier `ClockInterface` et `MockClock` (composant Clock installé) aux appels à l'heure réelle et à `sleep()`.
Une assertion d'absence d'appel est utile si le contrat dit « dry-run ne publie rien » ; une attente sur chaque getter ne l'est pas.

## 5. HTTP : le kernel fait partie du sujet

```php
use Symfony\Bundle\FrameworkBundle\Test\WebTestCase;

final class HealthTest extends WebTestCase
{
    public function testHealth(): void
    {
        $client = self::createClient();
        $client->request('GET', '/health');
        self::assertResponseIsSuccessful();
        self::assertResponseHeaderSame('content-type', 'text/plain; charset=UTF-8');
        self::assertSame('ok', $client->getResponse()->getContent());
    }
}
```

Ce fragment suppose une route `/health` créée pour cet exemple ; il ne prétend pas qu'elle existe dans le dépôt initial.
`createClient()` démarre le kernel : ne pas appeler `bootKernel()` juste avant dans le même test.
Le client exécute PHP en processus, sans serveur réseau ni moteur JavaScript ; `symfony serve -d` n'est pas requis pour ces tests.
Une requête JSON peut utiliser `jsonRequest()` ; vérifier statut, Content-Type et structure désérialisée.
Le client ne suit pas automatiquement toutes les redirections : inspecter `Location`, puis `followRedirect()` pour tester la destination.
Pour un hôte spécifique, fournir `['HTTP_HOST' => 'catalog.example.test']` dans les paramètres serveur de la requête.

## 6. Crawler et isolation entre requêtes

`request()` retourne un Crawler pour les documents HTML/XML ; `filter('h1')`, `text()`, liens et formulaires permettent des assertions sémantiques.
`assertSelectorTextContains()` évite une comparaison de toute la mise en page quand seul un titre appartient au contrat.
Un sélecteur absent retourne une sélection vide ; lire `text()` sans vérifier le contexte peut masquer le vrai problème.
BrowserKit ne clique pas un bouton piloté uniquement par JavaScript : ces scénarios exigent un autre niveau d'outillage.
Le client redémarre normalement le kernel entre requêtes ; un service modifié ou un double peut donc disparaître.
`disableReboot()` conserve le kernel mais des services peuvent être réinitialisés via leurs mécanismes `kernel.reset`.
Ne pas désactiver ce reset sans comprendre la fuite d'état que le test pourrait alors autoriser.
Chaque test crée ses fixtures et nettoie ses fichiers ; aucun ordre d'exécution ne doit être nécessaire.

## 7. Diagnostiquer un échec et profiler en option

Reproduire un seul test, lire la première erreur pertinente, puis réduire les entrées jusqu'au cas minimal.
Pour une erreur HTTP enveloppée, `$client->catchExceptions(false)` fait remonter l'exception au test.
Comparer la réponse réellement obtenue : une page d'erreur HTML n'est pas une erreur du parseur JSON client.
Lire `var/log/dev.log` pour l'environnement dev, `var/log/test.log` s'il est configuré pour test, puis le routage et le conteneur.
Le profiler est **absent** au départ. L'atelier facultatif demande `composer require --dev symfony/profiler-pack`, laissant Flex installer sa configuration.
Après installation, vérifier aussi la configuration de test : profiler disponible et collecte activable, éventuellement `collect: false` par défaut.
`$client->enableProfiler()` doit précéder la requête ; `$client->getProfile()` retourne ensuite un profil ou `false` s'il n'a pas été collecté.
Inspecter les collecteurs disponibles ; ne pas promettre de collecteur Doctrine dans ce dépôt sans Doctrine.
Une durée murale stricte est fragile en CI ; préférer des limites structurelles si le collecteur pertinent existe.

## 8. Commandes de travail et repères

```bash
php bin/phpunit tests/Testing
php bin/phpunit --filter testUnknownBookReturns404
php bin/phpunit --order-by=random tests/Testing
symfony console debug:router --env=test
symfony console lint:container --env=test
symfony console debug:container 'App\Testing\Catalogue' --env=test
```

Si `bin/phpunit` manque dans un autre projet : `vendor/bin/phpunit`. Installer les outils manquants avec `composer require --dev symfony/test-pack`.
Ici les outils nécessaires existent déjà ; ne pas confondre absence du métapaquet test-pack et absence de ses composants.
**Repère de versions :** les exemples de tests utilisent PHPUnit 13 et ses attributs ; ne pas reprendre les annotations PHPUnit d'un ancien cours Symfony 6/7.
**8.1 hors certification :** l'API résultat de `CommandTester::run()` est optionnelle ; ce module s'appuie sur `execute()` comme le [module Console](../13-console/README.md).
Voir [Testing Symfony 8.0](https://symfony.com/doc/8.0/testing.html), [DomCrawler](https://symfony.com/doc/8.0/components/dom_crawler.html) et [documentation PHPUnit 13.3](https://docs.phpunit.de/en/13.3/).
Vérifier `vendor/symfony/framework-bundle/Test/{KernelTestCase,WebTestCase}.php`, `KernelBrowser.php`, `phpunit.dist.xml` et `tests/bootstrap.php`.
