# Approfondissement — Téléchargement partiel et contexte du proxy

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Prérequis : requête/réponse, en-têtes et téléchargement du cours ; prévoir 40 minutes.
Périmètre : **Symfony 8.0**, comportements contrôlés sur **8.1.8**, sans composant supplémentaire.
Les exemples utilisent un fichier régulier et un proxy fictif de test, sans changer l'infrastructure réelle.

## 1. Une plage porte sur les octets d'une représentation

Un téléchargement interrompu peut reprendre par une requête `Range`.
Pour dix octets `abcdefghij`, `bytes=2-5` demande quatre octets, bornes incluses : `cdef`.
Les indices portent sur des octets, pas sur les caractères Unicode visibles.
Le serveur peut annoncer cette capacité avec `Accept-Ranges: bytes`.

`BinaryFileResponse` sait préparer cette réponse à partir d'un fichier dont la taille est connue.
Il calcule notamment le statut 206, `Content-Range` et la taille effectivement transférée.
Dans l'application, le listener de réponse appelle la préparation ; dans une expérience isolée, l'appeler explicitement.

```php
use Symfony\Component\HttpFoundation\BinaryFileResponse;
use Symfony\Component\HttpFoundation\Request;

$request = Request::create('/archive', server: ['HTTP_RANGE' => 'bytes=2-5']);
$response = new BinaryFileResponse($fixturePath);
$response->prepare($request); // Statut et en-têtes dépendent de la requête.
```

`$fixturePath` désigne un fichier contrôlé côté serveur, pas un chemin fourni directement par l'utilisateur.
La lecture du corps demande ensuite `sendContent()` ou la capture opérée par le client de test.
`getContent()` n'est pas une lecture du fichier.
Créer une nouvelle réponse par cas : `prepare()` modifie l'état qui contrôle l'envoi.

| Requête sur le fichier de dix octets | Résultat à observer |
|---|---|
| GET sans Range | 200, dix octets |
| GET `bytes=2-5` | 206, `Content-Range: bytes 2-5/10`, quatre octets |
| GET `bytes=-3` | 206, trois derniers octets |
| GET `bytes=20-30` | 416, `Content-Range: bytes */10` |
| HEAD | Métadonnées de téléchargement, aucun octet envoyé |

Limiter cet atelier à une plage valide unique ; le composant n'est pas présenté comme un encodeur de toutes les combinaisons multipart de la RFC.
Une plage invalide n'est pas synonyme de fichier absent : 416 et 404 décrivent des problèmes distincts.
L'autorisation d'accès au fichier reste obligatoire avant de retourner la réponse.

### `If-Range` évite d'assembler deux versions différentes

Le client possède le début d'une ancienne représentation et souhaite sa suite seulement si la version correspond.
Avec `Range` et un `If-Range` correspondant au validateur courant, la réponse partielle reste possible.
Si le validateur ne correspond plus, le serveur renvoie la représentation complète, normalement 200.
Ce résultat n'est ni un 304 ni un 412 : le client doit remplacer sa copie partielle.

Pour l'exercice, utiliser un ETag fort explicite, par exemple `setEtag('archive-v3')`.
La RFC exige une comparaison forte pour cette utilisation ; ne pas fabriquer un scénario fondé sur un ETag faible.
La comparaison installée se lit dans `BinaryFileResponse::hasValidIfRangeHeader()`.
Ne pas confondre cette vérification avec `Response::isNotModified()`, destiné à la validation de cache.

## 2. L'URL publique peut différer de la connexion reçue par PHP

Un proxy termine parfois HTTPS puis contacte PHP en HTTP sur le réseau interne.
Sans contexte supplémentaire, `Request::isSecure()` décrit cette connexion interne.
Une application peut alors générer une redirection HTTP alors que le navigateur utilisait HTTPS.
Les en-têtes `Forwarded` ou `X-Forwarded-*` transmettent des informations sur la requête publique.
Ils ne sont exploitables que si leur émetteur est reconnu et si chaque catégorie d'en-tête est autorisée.

La configuration a donc deux dimensions : adresses des proxies et masque des en-têtes acceptés.
Faire confiance à une adresse ne signifie pas accepter automatiquement son `X-Forwarded-Host`.
`trusted_hosts` constitue encore une autre règle : quels noms d'hôte l'application accepte.

```php
// Montage HttpFoundation dans un processus de test isolé.
Request::setTrustedProxies(
    ['192.0.2.10'],
    Request::HEADER_X_FORWARDED_FOR | Request::HEADER_X_FORWARDED_PROTO,
);
```

L'adresse appartient à une plage de documentation ; elle représente seulement le proxy fictif du test.
Pour l'application, les options de FrameworkBundle sont `trusted_proxies`, `trusted_headers` et `trusted_hosts`.
Configurer les valeurs correspondant à l'infrastructure réelle, sans copier l'adresse pédagogique en production.
Le proxy doit gérer correctement les en-têtes entrants avant de relayer ses propres valeurs.

| Situation | Interprétation attendue |
|---|---|
| Émetteur non approuvé + `X-Forwarded-Proto: https` | L'en-tête ne rend pas la requête sécurisée |
| Proxy approuvé + catégorie proto approuvée | `isSecure()` reflète le protocole transmis |
| Proxy approuvé + catégorie host non approuvée | Le nom d'hôte transmis n'est pas retenu |
| Host hors de `trusted_hosts` | Requête rejetée lorsque le host est validé |

`getClientIp()` calcule une adresse selon cette chaîne de confiance ; lire le premier champ brut n'est pas équivalent.
Cela concerne génération d'URL, cookies Secure et règles d'accès fondées sur l'adresse.
Ne pas réparer une mauvaise configuration en concaténant systématiquement `https://` dans les contrôleurs.
Le routeur et Twig doivent consommer un contexte de requête cohérent.

## Observer dans l'application

```bash
bin/console debug:config framework trusted_proxies
bin/console debug:config framework trusted_headers
bin/console debug:config framework trusted_hosts
php bin/phpunit tests/Http
```

Dans un test HttpFoundation, appeler vraiment `isSecure()`, `getHost()` et `getClientIp()` sur la requête construite.
Pour ces réglages statiques, isoler chaque test PHPUnit avec `#[RunInSeparateProcess]` et `#[PreserveGlobalState(false)]`, puis définir explicitement sa configuration.
Les deux attributs appartiennent à `PHPUnit\Framework\Attributes` ; dans ce montage de composant, ne pas démarrer un Kernel qui remplacerait les valeurs choisies.
`getTrustedHosts()` retourne des motifs déjà délimités : les repasser à `setTrustedHosts()` les enveloppe une seconde fois. Ce getter ne fournit donc pas une sauvegarde directement réinjectable.
Les getters de proxies et du masque d'en-têtes permettent une restauration ; pour les hosts, conserver les motifs bruts connus ou préférer l'isolation de processus proposée ici.
Pour le téléchargement, relever statut, taille des octets et `Content-Range` après préparation et envoi.
Lire `vendor/symfony/http-foundation/BinaryFileResponse.php` et les méthodes de confiance de `Request.php`.

## Documentation officielle

- [HttpFoundation — Symfony 8.0](https://symfony.com/doc/8.0/components/http_foundation.html#serving-files).
- [Configuration derrière un proxy — Symfony 8.0](https://symfony.com/doc/8.0/deployment/proxies.html).
- [Référence FrameworkBundle — Symfony 8.0](https://symfony.com/doc/8.0/reference/configuration/framework.html#trusted-hosts).
- [RFC 9110 : plages](https://www.rfc-editor.org/rfc/rfc9110.html#section-14) et [`If-Range`](https://www.rfc-editor.org/rfc/rfc9110.html#section-13.1.5).
