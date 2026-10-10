# Module 15 — HttpClient

[Parcours](../README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

## Objectifs et prérequis

Savoir appeler une API, distinguer une erreur HTTP d'une panne réseau, borner l'attente et tester sans réseau. Prérequis : [HTTP](../04-http-rfc9110/README.md), [DI](../03-dependency-injection/README.md) et [tests](../14-tests-diagnostic/README.md).

`symfony/http-client` est déjà installé. Les exercices utilisent un catalogue fictif sur `https://catalogue.example/` : aucun compte, secret ou service externe n'est nécessaire.

**Versions :** application Symfony 8.1.8 ; socle de certification Symfony 8.0. Les options récentes ne sont pas implicitement des prérequis du socle.

## 1. Dépendre du contrat

Injecte `Symfony\Contracts\HttpClient\HttpClientInterface` dans un service. Le contrôleur appelle ce service ; il ne connaît ni l'URL externe ni son protocole d'erreur. Utilise `query` pour les paramètres et `json` pour un corps JSON : évite de concaténer les paramètres ou d'encoder toi-même la requête.

Une réponse implémente `ResponseInterface`. `getStatusCode()` permet de décider explicitement comment interpréter un statut. `getContent()` lit un corps brut ; `toArray()` attend un JSON décodable en tableau. Un JSON valide peut rester invalide pour ton métier : il faut vérifier sa structure.

## 2. Réponses différées et exceptions

`request()` retourne une réponse dont la lecture peut attendre le réseau. Une erreur ne survient donc pas nécessairement au moment de cet appel : le `try` doit aussi couvrir les lectures.

| Famille | Situation | Décision à expliciter |
|---|---|---|
| `TransportExceptionInterface` | DNS, connexion, TLS, timeout | Indisponibilité de la dépendance |
| `HttpExceptionInterface` | Statut 300–599 lors d'une lecture avec exceptions activées | Interprétation du statut |
| `DecodingExceptionInterface` | JSON illisible ou non convertible en tableau | Réponse externe inexploitable |

Ces interfaces sont dans `Symfony\Contracts\HttpClient\Exception`. `getContent(false)` et `toArray(false)` désactivent les exceptions **de statut HTTP** ; ils ne rendent pas le réseau fiable et n'annulent pas les erreurs de décodage. Appeler `getStatusCode()` transfère au code appelant la responsabilité de traiter le statut ; cela ne rend pas un appel ultérieur à `toArray()` permissif.

## 3. Définir le périmètre d'un client

Un client nommé dans `framework.http_client.scoped_clients` centralise `base_uri`, les en-têtes et les délais d'un fournisseur. Vérifie son identifiant avec `debug:container`, puis cible-le explicitement avec `#[Autowire(service: 'catalogue.client')]` dans le constructeur si plusieurs clients existent.

La résolution des URL suit leurs règles habituelles : avec une base `https://catalogue.example/v1/`, `products` conserve `/v1/`, alors que `/products` repart à la racine. C'est un bon test de contrat pour un adaptateur externe.

`withOptions()` retourne un nouveau client : ne compte pas sur une modification de l'instance d'origine. Ne désactive pas la vérification TLS pour contourner une erreur de certificat.

## 4. Temps, reprises et effets de bord

`timeout` borne une période d'inactivité ; `max_duration` borne la durée de la requête et de la réponse dans leur ensemble. Un serveur peut continuer à émettre lentement et rester sous la limite d'inactivité tout en dépassant le budget total.

`RetryableHttpClient` délègue la décision de reprise à une stratégie. Décide séparément quels statuts sont temporaires et quelles opérations peuvent être répétées. Un POST de commande ne devient pas idempotent parce qu'un client le rejoue : il faut un contrat du serveur pour dédupliquer ses effets.

Dans les tests de reprise, simule une suite de réponses et configure un délai nul. Vérifie le nombre d'essais et la dernière erreur. N'impose pas une vraie attente au test.

## 5. Concurrence et streaming

Pour plusieurs appels indépendants, crée d'abord toutes les réponses, puis lis-les. Lire chaque réponse immédiatement dans la boucle empêche de profiter du chevauchement des requêtes.

`stream()` expose les réponses par morceaux. Identifie la réponse concernée et distingue premier morceau, contenu, fin et timeout. Une simulation valide l'assemblage et les erreurs ; elle ne prouve pas une accélération réseau réelle. Évite les assertions de durée sur une machine partagée.

## 6. Un double qui vérifie le contrat

Exemple minimal d'une réponse simulée, indépendant du service à réaliser :

```php
use Symfony\Component\HttpClient\MockHttpClient;
use Symfony\Component\HttpClient\Response\MockResponse;

$client = new MockHttpClient(new MockResponse('{"available":true}', ['http_code' => 200]));
$response = $client->request('GET', 'https://catalogue.example/status');
```

Une factory de réponse reçoit méthode, URL et options normalisées. Elle peut les vérifier avant de retourner un `MockResponse`. Une liste de réponses permet de représenter plusieurs appels. Utilise un nouveau client par test : une liste consommée n'est pas remise à zéro par magie.

Un test de service Symfony étend `KernelTestCase` : instancie le service avec ce double pour tester son comportement, puis ajoute un test de résolution par le container pour vérifier le câblage. Le module 14 explique quand un simple test d'objet suffit.

## Observer et diagnostiquer

```bash
php bin/console debug:autowiring HttpClientInterface
php bin/console debug:config framework http_client
php bin/console debug:container http_client
php bin/phpunit tests/HttpClient
```

Le dernier dossier sera créé par les exercices. Le profiler n'est pas installé dans le dépôt initial ; son ajout facultatif est présenté au module 14. Ne colle pas de token d'authentification dans un rapport de diagnostic.

## Complément 8.1 et sources

Le changelog local de HttpClient 8.1 mentionne notamment `max_connect_duration`. Compare cette borne de connexion à `timeout` et `max_duration` après avoir terminé le socle ; ne l'utilise pas dans une réponse portant explicitement sur 8.0.

- [HttpClient : documentation officielle](https://symfony.com/doc/current/http_client.html) — sélectionner la version étudiée.
- Sources locales : `vendor/symfony/http-client-contracts/ResponseInterface.php`, `vendor/symfony/http-client/MockHttpClient.php`, `vendor/symfony/http-client/CHANGELOG.md`.
- [Programme de certification](https://certification.symfony.com/exams/symfony.html).
