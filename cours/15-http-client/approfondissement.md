# Approfondissement — limiter un flux et borner une pagination

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md) · [Parcours](../README.md)

Le cours traite une réponse distante et les reprises ; un téléchargement ou une pagination ajoute d'autres limites.
Il faut décider combien de données accepter et à quel moment le résultat est complet.
HttpClient est installé et tous les laboratoires restent sans réseau grâce à `MockHttpClient`.
Les options et interfaces ci-dessous existent dans le socle Symfony 8.0.
`max_connect_duration`, propre au complément 8.1 du cours principal, n'est pas requis ici.

## 1. Streaming et consommation mémoire

Lire des morceaux avec `stream()` ne suffit pas à limiter la mémoire si tu les concatènes sans borne.
Le buffering interne du client et ton propre tampon sont deux mécanismes distincts.
`buffer: false` désactive le premier ; il ne supprime pas les chaînes que ton application conserve.
Compte les octets réellement consommés avec `strlen()`, en précisant que la limite porte sur ces octets.
Une longueur annoncée par le serveur n'est pas toujours disponible ni suffisante comme seule vérification.
Un budget de taille complète les budgets temporels `timeout` et `max_duration`.

```php
// $client implémente HttpClientInterface ; URL réservée au laboratoire simulé.
$response = $client->request('GET', 'https://catalogue.example/export', [
    'buffer' => false,
    'user_data' => 'export-demo',
]);
$identity = $response->getInfo('user_data');
```

`user_data` permet de retrouver le contexte local quand plusieurs réponses sont multiplexées.
Cette option n'est pas un paramètre envoyé dans le corps métier au fournisseur.
Dans la boucle de streaming, traite séparément timeout, arrivée des en-têtes, contenu et fin.
Le premier morceau n'est pas un morceau de données à concaténer aveuglément.
Un timeout de streaming laisse le choix de continuer ou d'annuler selon le contrat de l'application.
Ne considère pas la sortie de boucle après un abandon comme la preuve d'un document complet.
Voir [streaming HttpClient 8.0](https://symfony.com/doc/8.0/http_client.html#streaming-responses).

## 2. Annuler une réponse et préserver le contrat

`ResponseInterface::cancel()` arrête la consommation demandée au client.
Après cette annulation, `getInfo('canceled')` fournit une observation exploitable par le test.
Une annulation n'est pas une réponse vide réussie : le service doit annoncer son propre motif d'échec.
Une taille dépassée et une panne de transport sont deux causes qui peuvent demander des diagnostics différents.
Pour un téléchargement, ne retourne jamais un fichier final présenté comme complet après une interruption.
Dans un exercice en mémoire, l'absence de résultat partiel suffit à exprimer cette règle.
Dans un exercice sur fichier, prépare une destination temporaire puis publie le résultat seulement une fois complet.

Un fragment d'observation du protocole d'annulation :

```php
$response->cancel();
$cancelled = $response->getInfo('canceled');
```

Le test du service doit prouver pourquoi cette branche a été atteinte ; ces deux lignes seules ne le prouvent pas.
Le mock permet de fournir des morceaux connus, donc une frontière exacte et reproductible.
Son générateur est une simulation d'activités ; ne déduis pas une quantité exacte d'octets économisés sur un réseau réel.
La version installée prépare les activités du générateur dans `MockResponse::readResponse()`.
Évite donc de tester « le générateur n'a jamais avancé » pour démontrer l'annulation réseau.
Observe la réponse retournée par le client : elle n'est pas nécessairement le même objet que la fixture fournie au mock.
Les [interfaces de réponse](https://github.com/symfony/http-client-contracts/blob/v3.6.0/ResponseInterface.php) décrivent ce qui est garanti.

## 3. La pagination est un protocole applicatif

HttpClient récupère des réponses ; il ne connaît pas automatiquement le sens d'un champ JSON `next`.
Ton adaptateur doit définir les conditions de terminaison et d'échec de la collection.
Une page vide n'est pas forcément terminale si le protocole autorise encore une page suivante.
Inversement, `next: null` peut être terminal même si la page courante contient des produits.
Valide chaque page avant de publier l'ensemble : types de `items`, des identifiants et du pointeur suivant.
Un JSON syntaxiquement valide peut décrire un cycle : page 1 → page 2 → page 1.
Conserver les adresses déjà visitées permet de détecter ce cas avant de refaire le premier appel.
Une limite totale de pages borne aussi une suite infinie d'adresses toutes différentes.

```json
{"items":[{"id":71,"name":"Crayon"}],"next":"/v1/products?page=2"}
```

Cet exemple décrit un contrat d'atelier, pas un format de pagination imposé par Symfony.
Fixe les destinations autorisées avant de réutiliser la valeur distante comme URL.
Pour le laboratoire, une liste finie de chemins connus suffit à rendre cette décision testable.
Si une page intermédiaire échoue, décide si le service échoue entièrement ou expose explicitement un résultat incomplet.
Ne retourne pas silencieusement une liste amputée sous le même type qu'une liste complète.
Définis aussi le sens d'un identifiant répété : erreur, première occurrence ou dernière occurrence.
Ce choix influence l'ordre, les compteurs et les assertions ; le client HTTP ne le choisit pas pour toi.

| Situation | Décision du laboratoire | Preuve observable |
|---|---|---|
| Taille exactement à la limite | Accepter | Corps intégral exact |
| Taille supérieure à la limite | Annuler et échouer | `canceled` vrai, aucun résultat partiel |
| `next` vaut `null` | Terminer | Aucun appel supplémentaire |
| Pointeur déjà visité | Refuser le cycle | Pas de requête répétée |
| Page suivante hors contrat | Refuser avant appel | Factory du mock jamais sollicitée pour cette URL |
| Page invalide au milieu | Échec de l'opération | Aucun résultat présenté comme complet |

## Observer dans l'application

Après création des services des exercices :

```bash
php bin/console debug:autowiring HttpClientInterface
php bin/console debug:config framework http_client
php bin/phpunit tests/HttpClient
```

Conserve dans le double les méthodes, URL et options normalisées effectivement reçues.
Compare leur suite à la suite attendue ; le simple nombre de requêtes ne détecte pas un mauvais chemin.
Sources locales : `http-client/Response/MockResponse.php`, `http-client/MockHttpClient.php` et `http-client-contracts/ChunkInterface.php` sous `vendor/symfony/`.
La [documentation des réponses simulées 8.0](https://symfony.com/doc/8.0/http_client.html#testing) complète les exemples du cours.
Les nouveaux scénarios se trouvent dans les [exercices 9 et 10](exercices.md).
