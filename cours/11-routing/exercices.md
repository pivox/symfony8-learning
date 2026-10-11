# Exercices — Module 11 : mini-catalogue multilingue

[Parcours](../README.md) · [Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

Code : `src/Routing/`, tests : `tests/Routing/`. Routes en attributs, contrôleurs `AbstractController`, service catalogue en mémoire.
Choix du scénario : interface HTTP publique JSON, aucun stockage ni authentification. Les liens sont générés par le routeur.
Trois livres de fixture : `symfony`, `php`, `http`. Utiliser `WebTestCase` pour HTTP et `KernelTestCase` pour les services.
Chaque exercice demande les tests indiqués, pas une solution à recopier. Commande : `php bin/phpunit tests/Routing`.

## 1. Reconnaître un chemin — guidé

Scénario : exposer un livre avec `GET /lab/books/{slug}`, route `lab_book_show`, service tableau séparé.
Travail : écrire d'abord les cas HTTP, puis l'attribut et la délégation au catalogue.

- Attendus : `/lab/books/symfony` → 200 JSON avec `slug: symfony` ; `/lab/books/inconnu` → 404 métier.
- Acceptation : le slug inconnu reconnaît malgré tout `lab_book_show` dans `router:match` ; expliquer cette différence.
- Observation : `debug:router lab_book_show` montre chemin et méthode.
- Indice : le matcher ne consulte pas le tableau de livres ; la recherche métier commence ensuite.

## 2. Contraindre une pagination — guidé

Scénario : une liste accessible à `/lab/catalogue/{page}`, première page par défaut, trois livres par page.
Travail : choisir un requirement pour les entiers strictement positifs et un défaut cohérent.

- Attendus : `/lab/catalogue` et `/lab/catalogue/1` → même liste ; `/lab/catalogue/2` → liste vide 200.
- Acceptation : `/lab/catalogue/0`, `/lab/catalogue/-1`, `/lab/catalogue/abc` → 404 ; POST sur `/lab/catalogue/1` → 405.
- Observation : comparer le défaut et le requirement dans `debug:router` ; inspecter l'en-tête `Allow` du 405.
- Indice : un `int $page` ne remplace pas la contrainte de routage.

## 3. Réserver le mot search — semi-guidé

Scénario : `/lab/books/search?q=php` doit retourner une recherche, alors que `{slug}` accepterait aussi `search`.
Travail : provoquer la collision dans un test, puis la résoudre explicitement avec les attributs.

- Attendus : la recherche retourne une collection contenant `php` ; `/lab/books/http` reste une fiche unique.
- Acceptation : changer `q` conserve le nom de route ; tester `q=absent` → collection vide.
- Observation : conserver le résultat de `router:match /lab/books/search` avant et après correction.
- Indice : comparer priorité et ordre de déclaration ; la query ne départage pas deux chemins identiques.

## 4. Générer les liens — semi-guidé

Scénario : chaque entrée de liste expose un lien `self` et la liste un lien vers la page suivante.
Travail : injecter `UrlGeneratorInterface` dans un petit service de liens ; ne pas concaténer les chemins.

- Attendus : lien de `php` égal à `/lab/books/php` ; génération de page 2 avec filtre `theme=web` dans la query.
- Acceptation : nom inconnu et paramètre `slug` manquant provoquent les exceptions distinctes attendues.
- Cas limite : un slug incompatible avec ton requirement est rejeté en génération stricte ; tester la configuration réellement utilisée.
- Indice : les paramètres non consommés par le chemin peuvent finir dans la query string.

## 5. Ajouter deux langues — autonome

Scénario : les fiches deviennent `/fr/livres/{slug}` et `/en/books/{slug}` sous le nom logique `lab_book_localized`.
Travail : définir les chemins localisés et générer depuis le nom logique avec `_locale`.

- Attendus : français et anglais retournent le même identifiant de livre et indiquent la locale reconnue.
- Acceptation : génération explicite anglaise → `/en/books/php` ; `/de/books/php` → 404.
- Cas limite : le slug reste `php` dans les deux langues ; aucune traduction automatique du contenu n'est attendue.
- Indice : regarder les variantes produites dans `debug:router` avant d'inventer un suffixe de nom.

## 6. Faire varier l'hôte — autonome

Scénario : la version catalogue est désormais limitée à `catalog.example.test` ; les URL absolues servent aussi depuis une commande.
Travail : poser la contrainte d'hôte et tester le générateur dans un contexte maîtrisé.

- Attendus : bon chemin + bon hôte → 200 ; même chemin + `other.example.test` → 404.
- Acceptation : génération absolue contient `catalog.example.test` et le schéma choisi dans le contexte de test.
- Cas limite : restaurer le contexte après l'avoir modifié afin que les tests voisins n'héritent pas du faux hôte.
- Indice : KernelBrowser accepte `HTTP_HOST` ; aucun changement de DNS n'est nécessaire.

## 7. Diagnostiquer trois incidents — diagnostic

Scénario : un collègue signale une recherche absorbée par la fiche, un 405 inattendu et des liens anglais sortant en français.
Travail : recréer chaque défaut séparément dans ton code, écrire le test rouge minimal et établir sa cause.

- Attendus : une fiche de diagnostic par défaut avec requête complète, résultat observé, hypothèse et commande de vérification.
- Acceptation : les trois tests échouent avant correction, passent ensuite ; les fiches ordinaires restent accessibles.
- Preuves : ordre des routes pour la collision, `Allow` pour la méthode, paramètres `_locale` pour la génération.
- Indice : vérifier le contexte exact avant de vider les caches ; un cache n'explique pas systématiquement un mauvais paramètre.

## 8. Défi intégrateur — catalogue navigable

Scénario : livrer la liste, recherche, pagination et fiches bilingues sur l'hôte du catalogue, entièrement reliées par URL générées.
Travail autonome : harmoniser noms de routes, représentation JSON et service de liens ; documenter les règles dans un court tableau de cas.

- Attendus : partir de la liste française, suivre un lien de fiche, générer sa variante anglaise et obtenir le même livre.
- Acceptation : tests 200, 404 de routage, 404 métier, 405, recherche vide, locale inconnue et hôte incorrect.
- Frontière : query additionnelle conservée comme query, jamais utilisée comme autorisation ; aucun URL construit manuellement.
- Diagnostic final : expliquer un échec de génération distinct d'un échec de reconnaissance avec leurs exceptions/statuts.
- Indice : assembler les comportements des exercices précédents ; un test de parcours protège leur cohérence.

## 9. Un même nom dans le chemin et la query — intermédiaire

Durée indicative : 35–50 min. Prérequis : génération, requête HTTP et [parties d'une URL](approfondissement.md).
Scénario : `GET /lab/editions/{slug}` identifie l'édition affichée ; la query `slug` indique une édition à comparer.
Créer la route nommée `lab_edition_show` en attribut, avec réponse JSON exposant séparément `pathSlug` et `querySlug`.
Dans un service de liens, générer une URL avec chemin `php`, query `slug = symfony`, query `q = 'a & b'` et fragment `comparison`.

- Réussite : chemin généré `/lab/editions/php`, query décodée contenant `slug = symfony` et `q = 'a & b'`, fragment exact `comparison`.
- Réussite HTTP : appeler chemin + query → 200, `pathSlug = php`, `querySlug = symfony` ; aucun fragment n'est nécessaire dans l'appel serveur.
- Cas négatif : transmettre une chaîne au lieu d'un tableau pour `_query` → exception de génération vérifiée ; route inconnue → `RouteNotFoundException` dans un test séparé.
- Comparaison : fournir `q = ignored` comme paramètre supplémentaire et `q = 'a & b'` dans `_query`, puis constater la priorité de la query explicite.
- Preuve : appel au générateur réellement injecté, puis requête `WebTestCase` sur l'URL générée sans fragment ; aucun `urlencode()` préalable.
- Indice 1 : les attributs et la query de `Request` sont deux sacs distincts même lorsque les noms de clés coïncident.
- Indice 2 : `_query` permet de conserver un nom déjà consommé par le chemin ; `_fragment` appartient au navigateur.

## 10. Reconnaître la réponse avant une redirection — avancé

Durée indicative : 45–60 min. Prérequis : méthodes HTTP et [HEAD et canonicalisation](approfondissement.md).
Scénario : une action de diagnostic publique est déclarée exactement sur `/lab/canonical-report`, avec méthodes GET et POST.
Elle renvoie le texte fixe `report-ready` et l'en-tête `X-Report-Version: 1` ; aucun effet métier ni stockage.
Tester séparément GET, HEAD, POST et le chemin avec slash final, sans suivre automatiquement les redirections.

- Réussite : GET canonique → 200, corps exact et header ; HEAD canonique → 200, même header et corps vide.
- Réussite : POST canonique → 200 ; GET `/lab/canonical-report/?mode=short` → 301 vers la forme sans slash, query `mode=short` conservée.
- Réussite : suivre explicitement cette redirection → 200 avec `report-ready` ; vérifier d'abord la réponse 301 pour ne pas la masquer.
- Cas négatif : POST avec slash final → 404, sans redirection ; DELETE sur le chemin canonique → 405 avec GET et POST dans `Allow`.
- Diagnostic : expliquer pourquoi un `UrlMatcher` autonome et le routeur HTTP du framework ne prouvent pas les mêmes comportements de redirection.
- Indice 1 : HEAD est reconnu avec GET, puis le corps de réponse est supprimé au niveau HTTP.
- Indice 2 : lire la condition sur GET/HEAD dans le matcher redirigeant ; tester les statuts avec `KernelBrowser`, pas seulement `router:match`.
