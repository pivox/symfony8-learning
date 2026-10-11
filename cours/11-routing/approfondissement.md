# Approfondissement — URL complète, redirections et méthode HEAD

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Le cours établit le matching et la génération ; cette fiche examine leurs frontières HTTP.
Socle **Symfony 8.0**, vérifié dans **8.1.8** ; pas d'option spécifique à 8.1 nécessaire.
Prérequis : route en attributs, contexte du générateur, `Request` et tests avec `KernelBrowser`.
Les exemples restent publics et en mémoire, sans couche de persistence.

## 1. Le chemin, la query et le fragment ont des propriétaires distincts

Dans `/files/report?download=1#details`, le matcher compare le chemin `/files/report`.
La query `download=1` reste disponible pour le contrôleur ou un DTO de query.
Le fragment `details` sert au navigateur ; il n'est pas envoyé dans la requête HTTP au serveur.
Une action ne peut donc pas sélectionner une représentation à partir d'un fragment reçu du navigateur.
Le générateur sait produire ces trois parties, mais elles ne jouent pas le même rôle au retour.

```php
// La route document_show possède un placeholder {slug}.
$url = $urls->generate('document_show', [
    'slug' => 'guide',
    '_query' => ['slug' => 'comparison', 'mode' => 'print'],
    '_fragment' => 'examples',
]);
```

Le `slug` de chemin et le `slug` de query peuvent ainsi coexister sans renommer le placeholder.
À la reconnaissance, lire le premier dans `Request::attributes`, le second dans `Request::query`.
La clé `_query` doit contenir un tableau ; une chaîne arbitraire n'est pas une query déjà validée.
Les paramètres supplémentaires peuvent aussi produire de la query sans `_query`.
Si les deux voies fournissent la même clé supplémentaire, les valeurs explicites de `_query` prennent le dessus.
Cette priorité ne modifie pas la valeur nécessaire au chemin.

### L'encodage appartient au générateur

Fournir des valeurs de paramètres ordinaires, sans appliquer `urlencode()` avant génération.
Une valeur déjà encodée risque d'être encodée à nouveau, notamment le caractère `%`.
Les règles ne sont pas identiques pour un segment de chemin, une query et un fragment.
Un espace dans une valeur de query est un bon témoin ; des slashs dans un placeholder demandent un requirement adapté.
Le résultat du générateur n'assure toujours pas que la ressource métier existe.

| Élément | Provenance à la génération | Observation après requête |
|---|---|---|
| placeholder `slug` | paramètre `slug` | attribut de route |
| query `slug` du même nom | `_query['slug']` | paramètre de query distinct |
| query de recherche | paramètre supplémentaire ou `_query` | query décodée |
| fragment | `_fragment` | absent de la requête serveur |

Dans un test, analyser les parties de l'URL peut être plus robuste que figer l'ordre de toutes les clés de query.
En revanche, le chemin, les valeurs décodées et le fragment doivent correspondre exactement au contrat.
Éviter une assertion « contient guide » qui réussirait si `guide` n'apparaissait que dans la mauvaise partie.

## 2. Une URL générée peut devoir changer de schéma ou d'hôte

Le mode `ABSOLUTE_PATH` demande normalement un chemin, mais les contraintes de la route comptent aussi.
Si la route exige HTTPS et que le contexte courant est HTTP, le générateur doit exprimer le changement de schéma.
Il peut donc produire une URL absolue, même pour une demande initiale de chemin.
Si l'hôte de la route diffère, une référence commençant par `//` peut être nécessaire.
Le contexte du routeur participe au résultat ; un test qui le modifie doit le restaurer ensuite.

```php
use Symfony\Component\Routing\Attribute\Route;

#[Route('/secure-report', name: 'secure_report', schemes: ['https'], methods: ['GET'])]
// Attribut à placer sur une action ; le nom est distinct des ateliers.
```

Une contrainte de schéma n'installe pas de certificat TLS ni de serveur HTTPS.
Dans `WebTestCase`, l'environnement serveur de la requête permet de simuler le schéma sans réseau réel.
Un test de génération depuis un contexte HTTP puis HTTPS explique pourquoi la forme du résultat varie.
Ce réglage ne remplace pas les contrôles Security : une URL HTTPS peut rester publique.

## 3. HEAD et slash final : tester la réponse intermédiaire

Une route autorisant GET peut être reconnue pour HEAD.
Le contrôleur peut être exécuté, tandis que la préparation de la réponse HTTP retire son corps pour HEAD.
HEAD n'est donc pas la preuve qu'aucun service ou calcul n'a été exécuté.
Une action GET ne devrait pas modifier l'état métier simplement parce qu'elle est consultée.
Tester un compteur d'observation est utile ; déclencher une écriture métier pour fabriquer une preuve ne l'est pas.

Avec le routeur FrameworkBundle, une différence de slash final peut conduire à une redirection canonique.
Ce comportement de redirection concerne GET et HEAD ; ne pas extrapoler à POST ou PATCH.
Le matcher autonome `UrlMatcher` n'a pas à lui seul toute l'intégration de redirection HTTP du framework.
Il peut donc lever une exception là où une requête applicative reçoit une réponse de redirection.
Conserver les deux niveaux dans des tests séparés évite de comparer des contrats différents.

| Appel | Ce qu'il faut relever | Piège évité |
|---|---|---|
| GET canonique | statut, corps et headers | tester seulement une route enregistrée |
| HEAD canonique | statut, headers et corps vide | supposer le contrôleur ignoré |
| GET avec slash incorrect | statut de redirection et `Location` | suivre automatiquement et masquer l'étape |
| POST avec slash incorrect | statut réellement refusé | supposer une redirection universelle |
| HTTP vers route HTTPS | réponse et schéma cible | assimiler contrainte à configuration TLS |

Dans un test navigateur, commencer sans suivi automatique des redirections.
Vérifier la première réponse, puis suivre explicitement le lien si le scénario l'exige.
Une query utile doit être contrôlée après canonicalisation, pas seulement le chemin cible.
Une redirection inattendue peut expliquer qu'un test trouve du HTML au lieu du JSON attendu.

## Observer dans l'application

```bash
symfony console debug:router
symfony console router:match /secure-report --method=HEAD --scheme=https
symfony console debug:config framework router
php bin/phpunit tests/Routing
```

La commande d'exemple suppose l'action illustrative créée ; adapter le chemin aux routes de l'atelier.
Les statuts et `Location` se vérifient par requête HTTP ; `router:match` ne remplace pas ce test.
En CLI, injecter `UrlGeneratorInterface` pour les appels de génération et maîtriser son `RequestContext`.
Lire `var/log/dev.log` pour distinguer un refus du matcher d'une exception déclenchée dans l'action.

Sources : [Routing 8.0](https://symfony.com/doc/8.0/routing.html), [HttpFoundation 8.0](https://symfony.com/doc/8.0/components/http_foundation.html).
Sources locales : `routing/Generator/UrlGenerator.php`, `routing/Matcher/RedirectableUrlMatcher.php`, `routing/Matcher/Dumper/CompiledUrlMatcherTrait.php`.
Compléter par `framework-bundle/Controller/RedirectController.php` et `http-foundation/Response.php`, méthode `prepare()`.
Relier enfin les exercices au [module HTTP](../04-http-rfc9110/README.md) : méthode, statut et représentation constituent un seul contrat observable.
