# Approfondissement — temps déterministe et parcours avec cookies

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Un test fiable contrôle les états qui influencent le résultat, même quand ils semblent invisibles.
Deux exemples fréquents sont l'heure courante et les cookies conservés entre requêtes.
Clock et BrowserKit sont installés ; ces laboratoires ne demandent ni base ni authentification.
Les mécanismes présentés sont disponibles dans le socle Symfony 8.0 et dans le dépôt 8.1.8.
Les règles temporelles et les routes des exemples restent des contrats d'atelier à créer.

## 1. Tester l'instant frontière

« Valide pendant une minute » laisse une question ouverte : que se passe-t-il exactement à l'échéance ?
Écris d'abord la règle mathématique, par exemple valide si `maintenant < expiration`.
Les cas juste avant, exactement à et juste après l'échéance deviennent alors trois assertions distinctes.
Le choix strict ou inclusif appartient au métier ; l'horloge ne le décide pas.
Injecte `Symfony\Component\Clock\ClockInterface` dans le service qui prend cette décision.
Un appel direct à `new DateTimeImmutable()` dans ce service contournerait l'horloge injectée.

```php
use Symfony\Component\Clock\MockClock;

$clock = new MockClock('2031-04-05T12:00:00+00:00');
$before = $clock->now();
$clock->sleep(15);
$after = $clock->now();
```

Ici, `sleep()` avance l'horloge simulée sans suspendre le test pendant quinze secondes.
Il ne change pas l'horloge du système ni les appels natifs à `time()` ailleurs dans le programme.
Les deux objets date représentent des instants distincts, même si le test se termine immédiatement.
Choisis une date et un fuseau explicites pour ne pas dépendre de la machine qui exécute PHPUnit.
Une date affichée dans un fuseau local et l'instant comparé en UTC peuvent décrire le même événement.
Teste la règle sur les instants ; ajoute un test de formatage seulement si l'affichage fait partie du contrat.
La [documentation Clock 8.0](https://symfony.com/doc/8.0/components/clock.html) distingue ces usages.

Dans un `KernelTestCase`, remplace l'horloge réellement injectée avant de récupérer le consommateur.
Vérifie cet identifiant avec `debug:autowiring ClockInterface` et les arguments du service.
Changer une autre horloge dans le test ne suffit pas à contrôler celle du service.
Après une récupération du consommateur, remplacer sa dépendance dans le conteneur arrive trop tard.
Pour une série de requêtes HTTP, le reboot du kernel peut aussi reconstruire l'horloge réelle.
Un test indépendant par instant frontière est souvent plus simple qu'un double maintenu entre requêtes.
Le [cours sur les tests Symfony 8.0](https://symfony.com/doc/8.0/testing.html) détaille le remplacement des services.

## 2. Le navigateur et le kernel n'ont pas la même durée de vie

BrowserKit possède un historique et un `CookieJar` indépendants des services de l'application.
Le kernel peut être redémarré entre deux requêtes alors que le navigateur conserve ses cookies.
C'est précisément ce qui permet de tester un parcours utilisateur sur plusieurs pages.
Un cookie reçu dans `Set-Cookie` est enregistré puis renvoyé aux URL qui correspondent à sa portée.
Cette portée inclut notamment le domaine, le chemin et le caractère sécurisé du cookie.
Un cookie limité à `/lab/testing` n'a pas vocation à être envoyé à `/autre`.
Un cookie `Secure` demande une URL HTTPS dans le parcours qui prétend le transmettre.
Ces propriétés se testent via de vraies requêtes du client, pas uniquement via un getter de `Cookie`.

```php
// Dans un WebTestCase, après une réponse ayant posé un cookie :
$cookies = $client->getCookieJar()->all();
$client->restart();
// Le prochain parcours n'hérite plus des cookies ni de l'historique précédents.
```

`restart()` n'est pas une remise à zéro générale de tous les objets PHP de ton test.
Pour les services, c'est le cycle du kernel qui reste pertinent.
Pour les fichiers temporaires, c'est le nettoyage explicite de la fixture qui reste pertinent.
Confondre ces trois états peut produire un test qui passe seulement après une autre méthode.
La [documentation BrowserKit 8.0](https://symfony.com/doc/8.0/components/browser_kit.html#cookies) présente le cookie jar.

## 3. Vérifier une redirection en deux étapes

Un statut de redirection et une page finale réussie prouvent deux comportements différents.
Après un POST valide, vérifie d'abord le statut et l'en-tête `Location` sans suivi automatique.
Appelle ensuite `followRedirect()` et vérifie le contenu ainsi que la méthode de la nouvelle requête.
Une redirection 303 exprime un retour vers une consultation ; le client effectue ici un GET.
Une 307 ou 308 conserve la méthode et le corps : ce n'est pas interchangeable dans un parcours POST.
Le cas invalide doit prouver l'absence du changement d'état prévu, par exemple aucun nouveau cookie.
Une réponse 422 avec un cookie malgré tout peut révéler un effet exécuté avant la validation.
Garde le contrôleur mince : le test HTTP prouve que validation et orchestration sont effectivement branchées.

| État ou transition | Ce qui le contrôle | Preuve utile |
|---|---|---|
| Heure métier | Horloge injectée | Résultat exact avant et à l'échéance |
| Cookie du navigateur | `CookieJar` | Valeur reçue après navigation |
| Redirection | Statut et `Location` | Destination puis méthode réellement utilisée |
| Services entre requêtes | Reboot ou reset du kernel | Nouvelle résolution des dépendances |
| Fichier produit | Fixture temporaire | Existence, contenu puis suppression |

## Observer et reproduire

Après réalisation des services et routes demandés :

```bash
php bin/console debug:autowiring ClockInterface --env=test
php bin/console debug:router --env=test
php bin/phpunit --filter Expiration tests/Testing
php bin/phpunit --filter Preference tests/Testing
php bin/phpunit --order-by=random tests/Testing
```

Commence par une assertion qui compare le résultat métier exact.
Ajoute ensuite l'assertion qui détecte l'effet de bord interdit dans le cas négatif.
Une suite aléatoire ne prouve pas l'indépendance universelle, mais elle révèle des dépendances cachées.
Relis `vendor/symfony/clock/MockClock.php` pour le fonctionnement de `sleep()`.
Relis `vendor/symfony/browser-kit/AbstractBrowser.php`, méthodes `followRedirect()` et `restart()`.
Relis `vendor/symfony/framework-bundle/KernelBrowser.php` pour distinguer navigateur et kernel.
Ces observations ne nécessitent pas le profiler, absent du dépôt initial.
