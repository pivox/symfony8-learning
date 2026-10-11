# Apprendre, vérifier et diagnostiquer

[Parcours](README.md) · [Ateliers transversaux](ateliers/README.md) · [Module Tests](14-tests-diagnostic/README.md)

Cette fiche accompagne les exercices. Elle sert à choisir une preuve, organiser une séance et retrouver l’origine d’un comportement surprenant. Les durées sont des repères de travail personnel, pas une estimation de ton niveau.

## Une séance en quatre temps

| Temps indicatif | Travail | Trace à conserver |
|---|---|---|
| 10 minutes | Expliquer une notion sans ouvrir le cours | Une hypothèse et un exemple |
| 15 minutes | Lire le passage utile et observer l’API installée | Version, commande ou fichier source |
| 30–60 minutes | Réaliser un exercice et ses cas limites | Tests avec assertions de résultat |
| 10 minutes | Expliquer ce qui a changé dans ta compréhension | Une règle et une erreur à ne plus reproduire |

Si une séance s’arrête au premier test rouge, noter l’entrée, le résultat attendu et le résultat réel suffit pour reprendre. Ne pas marquer l’exercice terminé parce que le cas nominal fonctionne : reprendre sa liste de critères.

Pour revenir sur une notion, tente un exemple différent le lendemain, puis quelques jours plus tard. Un quiz déjà mémorisé peut donner une impression de maîtrise ; changer les données permet de vérifier le raisonnement.

## Lire un énoncé comme un contrat

Avant de coder, écrire cinq phrases :

1. **Entrée :** quelle valeur arrive, par quel appel et sous quel type ?
2. **Précondition :** quel état initial le test prépare-t-il ?
3. **Sortie :** quel résultat exact le demandeur reçoit-il ?
4. **Effet :** quel message, fichier, email ou changement est autorisé ?
5. **Échec :** quelle entrée doit être refusée, et quel effet doit rester absent ?

Exemple de formulation : « Pour un catalogue contenant les identifiants 41 et 42, la recherche de 99 retourne une absence ; elle ne crée pas automatiquement un produit. » Le choix `null`, exception ou HTTP 404 dépend ensuite de la frontière étudiée.

Les fixtures appartiennent au test : pas de dépendance à la date du jour, à une API publique, à une boîte email ou à l’ordre des tests. Les fichiers générés vont dans un répertoire temporaire propre au test. Les classes placées sous `tests/` utilisent `App\Tests\...`, celles sous `src/` utilisent `App\...`.

## Choisir la preuve qui répond à la question

| Question | Preuve adaptée | Ce qui reste à vérifier ailleurs |
|---|---|---|
| La valeur refuse-t-elle une quantité négative ? | Appel du constructeur ou de la méthode, puis assertion précise | Mapping HTTP |
| Le bon service est-il injecté ? | `KernelTestCase`, appel du consommateur obtenu dans le conteneur | Présentation HTML/JSON |
| Une requête invalide est-elle rejetée ? | `WebTestCase`, statut, contenu et absence d’effet | Réseau d’un serveur déployé |
| La commande respecte-t-elle ses options ? | Test Console, statut, stdout/stderr et effets | Supervision du processus |
| Un message a-t-il été mis en file ? | Dispatch puis inspection du transport en mémoire | Consommation et acquittement |
| Le traitement métier fonctionne-t-il ? | Appel du handler et inspection de ses effets simulés | Retry piloté par le worker |
| Le cache évite-t-il un recalcul ? | Deux lectures et compteur sur la source | Cache partagé entre machines |

Le projet utilise `KernelTestCase` pour les services et `WebTestCase` pour HTTP. Un objet pur peut être testé sans kernel. Ne pas remplacer toutes les dépendances par des mocks : garder le mécanisme étudié réel, et doubler sa frontière externe.

Un test qui attend seulement une classe d’exception peut réussir pour la mauvaise raison. Vérifier aussi la valeur, le chemin de violation ou la cause lorsqu’ils font partie du contrat. Un statut HTTP 200 sans vérification du contenu peut cacher une page de connexion ou une réponse vide.

## Fiche d’incident reproductible

À garder dans tes notes personnelles pour chaque diagnostic :

```text
Module / exercice :
Version PHP, Symfony et environnement :
Commande ou requête exacte :
Entrée et état initial :
Résultat attendu :
Résultat obtenu :
Première cause observée (fichier, exception, route ou service) :
Hypothèse vérifiable :
Expérience minimale :
Correction et test qui aurait détecté le défaut :
```

Faire varier un seul facteur à la fois : méthode HTTP, hôte, donnée, environnement ou dépendance. Si tu modifies quatre réglages ensemble, un test vert ne dit plus lequel a corrigé l’incident.

## Retrouver la bonne couche

| Symptôme | Première observation | Ensuite |
|---|---|---|
| 404 avant l’action | `debug:router`, puis `router:match` avec méthode et hôte | Requirement, ordre des routes, environnement |
| 404 après entrée dans l’action | Catalogue/fixture et exception métier | Test du service puis HTTP |
| 405 | Méthode reçue et en-tête `Allow` | Attribut `#[Route]` et contrat du client |
| 400/415/422 sur un payload | `Content-Type`, corps exact, attribut de mapping et violations | Séparer JSON invalide, format non accepté et données invalides |
| 401/403 ou redirection inattendue | Firewall correspondant, entry point, token et décision | Règle d’accès et voter ; suivre le vrai parcours d’authentification |
| Argument de service introuvable | `debug:autowiring`, alias et argument du consommateur | Enregistrement et environnement du service |
| Message accepté, aucun effet | Routage Messenger et transport sélectionné | Dispatch, consommation puis handler, dans cet ordre |
| Valeur ancienne après écriture | Clé, pool, TTL et invalidation | Distinguer cache applicatif et cache HTTP |
| Test vert seul, rouge en suite | Fichiers, variables globales, horloge, services partagés | Rejouer avec ordre aléatoire et seed conservée |
| Traduction non appliquée | Locale, domaine, identifiant et catalogue | Présence du composant, repli, paramètres ICU |

Un 422 n’est pas universel : `MapRequestPayload` et `MapQueryString` n’ont pas les mêmes valeurs par défaut. Vérifie l’attribut et le contrat de l’exercice avant de modifier l’attente.

## Commandes à adapter à ton exercice

```bash
symfony console about
symfony console debug:router --env=test
symfony console debug:autowiring --env=test
symfony console debug:event-dispatcher kernel.request --env=test
symfony console debug:config framework --env=test
symfony console lint:container --env=test
php bin/phpunit --filter testNomDuScenario
php bin/phpunit --order-by=random --random-order-seed=42
```

Remplace `testNomDuScenario` par le nom d’un test écrit. La seed aide à reproduire un ordre ; elle ne réinitialise pas les services ou les fichiers. Si la CLI Symfony n’est pas disponible, utiliser `php bin/console`.

Lire `var/log/dev.log` lorsque le journal existe ; dans un autre environnement, vérifier sa configuration réelle. Le profiler n’est pas installé dans le socle de ce dépôt : l’ajouter avec `composer require --dev symfony/profiler-pack` au moment d’en avoir besoin. Son absence n’empêche ni les tests ni l’inspection des exceptions.

## Demander un indice utile

Fournis le numéro de module/exercice, le résultat attendu, le résultat réel et le plus petit extrait qui reproduit l’écart. Demande d’abord une direction, puis une API ou une commande d’observation si nécessaire. Une trace complète peut contenir des valeurs locales : remplacer les secrets par des valeurs fictives avant de la partager.

Une bonne explication de fin d’exercice répond à trois questions : quelle couche décide, comment l’observer, et quel test détecte une régression ? Pour un approfondissement, ajoute une limite : ce que ton expérience ne prouve pas encore.

## Grille d’autoévaluation

| Niveau | Preuve personnelle |
|---|---|
| Repérer | Je retrouve l’API dans la version installée et je nomme sa responsabilité |
| Appliquer | Je réussis le cas nominal et une frontière avec des assertions précises |
| Diagnostiquer | Je reproduis un défaut sans le masquer et j’identifie sa cause |
| Transférer | Je réussis un atelier associant plusieurs modules et j’explique les limites |

Cette grille complète les cases du parcours ; elle ne les coche pas automatiquement. Un atelier peut révéler un prérequis à reprendre sans annuler les connaissances déjà acquises.

Références : [tests Symfony 8.0](https://symfony.com/doc/8.0/testing.html), [bonnes pratiques](https://symfony.com/doc/8.0/best_practices.html), [options PHPUnit 13.3](https://docs.phpunit.de/en/13.3/textui.html). Les contrats exacts se vérifient aussi dans les classes installées sous `vendor/`.
