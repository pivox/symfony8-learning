# Exercices — Module 03

Code dans `src/Di/` (namespace `App\Di`), tests dans `tests/Di/` (namespace `App\Tests\Di`) : `php bin/phpunit tests/Di`. Écris le test d'abord. Scénario fil rouge : un petit système de **notification** (`Notifier`) avec plusieurs canaux (`ChannelInterface` : email, sms, slack...).

## 1. Observer le container
Sans coder : avec `debug:container --tag=kernel.event_listener`, `debug:autowiring Logger`, `debug:container --types`, repère (a) un service privé, (b) un alias d'interface, (c) un tag avec attributs.
- Questions : pourquoi une définition visible avec `debug:container` peut-elle être inaccessible à l’exécution ? Où est dumpé le container compilé ?

## 2. Autoconfigure + tag + tagged iterator
Crée `ChannelInterface` annotée `#[AutoconfigureTag('app.notification_channel')]`, deux implémentations, et un `Notifier` qui reçoit `#[AutowireIterator('app.notification_channel')] iterable $channels`.
- Tests (KernelTestCase) : le `Notifier` itère sur les deux canaux ; ajouter un troisième canal sans toucher à la config le fait apparaître.
- Questions : l'itérateur est-il un `array` ? Les canaux sont-ils instanciés avant l'itération ?

## 3. Priorité et index
Ajoute `#[AsTaggedItem(index: 'sms', priority: 20)]` aux canaux. Passe de l'itérateur à un `#[AutowireLocator('app.notification_channel')] ServiceLocator`.
- Tests : l'ordre d'itération suit la priorité ; `$locator->get('sms')` retourne le bon canal ; `has('inconnu')` est faux.
- Réflexion : qu'est-ce qui s'est passé avec `defaultIndexMethod` en 8.1 ? Pourquoi l'itérateur est-il préférable ici (ou le locator, selon le cas d'usage) ?

## 4. Compiler pass maison
Écris `RegisterChannelsPass` (`CompilerPassInterface`) qui lit les services taggés `app.notification_channel` et les ajoute via `addMethodCall` à un `ChannelRegistry` (sans utiliser les attributs d'itérateur).
- Test **unitaire** (pas de Kernel) : `ContainerBuilder` + `register()` + `addCompilerPass()` + `compile()` ; vérifie les `MethodCall`.
- Enregistrement dans l'app : via `Kernel::build()`. Teste en test d'intégration que ça marche.
- Questions : à quelle phase de `PassConfig` ta passe s'exécute-t-elle par défaut ? Que changerait `TYPE_BEFORE_REMOVING` ? Pourquoi ne peux-tu pas lire un service privé en fin de compilation ?

## 5. Décoration
Crée `LoggingChannel` décorant un canal via `#[AsDecorator(decorates: SmsChannel::class)]` et `#[AutowireDecorated]`. Ajoute un second décorateur `RetryChannel` avec une autre priorité.
- Tests : l'ordre d'appel (log puis retry, ou l'inverse) correspond à tes priorités ; l'id d'origine renvoie le décorateur le plus externe.
- Questions : comment s'appelle le service interne (`.inner`) ? Quel décorateur est le plus interne : priorité la plus haute ou la plus basse ? Vérifie avec `debug:container` (pile de décoration) et en lisant `DecoratorServicePass`.

## 6. Factory et `#[Autowire]`
Crée `HttpClientFactory` (service) avec une méthode `create(string $baseUri)` qui produit un objet `SlackClient`. Câble-le via `services.yaml` (`factory:`), le `$baseUri` venant d'un **paramètre** puis d'une **variable d'environnement** via `#[Autowire(env: '...')]`.
- Tests : le client reçoit le bon `baseUri` (en test, `.env.test` ou `$_ENV`).
- Réflexion : factory statique vs factory service ; quand utiliser `#[AutowireServiceClosure]` ou `#[AutowireCallable]` plutôt qu'une factory ?

## 7. Lazy
Crée un service `ExpensiveReportGenerator` dont le constructeur incrémente un compteur statique. Injecte-le avec `#[Lazy]` dans un service qui ne l'utilise que rarement.
- Tests : construire le service consommateur n'incrémente **pas** le compteur ; appeler une méthode utilisant le générateur l'incrémente une fois.
- Questions : que se passe-t-il pour une classe `final` ? Comment ceci se relie-t-il aux lazy objects PHP 8.4 du module 01 ? Où voir le code généré dans `var/cache/` ?

## 8. `#[When]`, `#[Target]` et ServiceSubscriber
(a) `FakeSmsChannel` avec `#[When(env: 'test')]` et la vraie `SmsChannel` avec `#[WhenNot(env: 'test')]`, tous deux alias de `ChannelInterface` nommé (via `#[Target]` ou `#[AsAlias]`).
(b) Un `ReportBuilder` implémentant `ServiceSubscriberInterface` qui ne reçoit qu'un locator vers `LoggerInterface` et `Environment` (Twig).
- Tests : en `test`, c'est le Fake qui est injecté ; le locator du `ReportBuilder` ne contient que les services listés (`getProvidedServices()`), `has(Autre::class)` faux.
- Réflexion : pourquoi `AbstractController` est-il un *service subscriber* ? Compare avec `#[AutowireLocator([...])]`.

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.


## Critères complémentaires des exercices 1–8

- **1** : distinguer définition visible dans le dump et service récupérable à l'exécution ; consigner environnement et identifiant exact.
- **2–3** : deux parcours de l'itérable donnent les mêmes canaux dans le même ordre ; un locator inconnu a `has() = false`, son `get()` échoue. Fixer des priorités différentes pour ne pas dépendre d'une égalité.
- **4** : dans le test de passe, conserver une définition de registre observable (par exemple publique dans le `ContainerBuilder` isolé) ; sinon l'optimisation peut la supprimer. Vérifier aussi le résultat d'un appel au registre intégré au Kernel.
- **5** : tester succès **et** exception interne ; identifier le `.inner` du décorateur, pas un nom supposé fondé sur le service d'origine.
- **6–8** : aucune connexion réseau réelle ; vérifier la valeur injectée et le canal de test. Pour le lazy, appeler une méthode qui lit réellement l'état. Pour la factory, réserver YAML à ce câblage explicite et garder les paramètres scalaires en attribut `#[Autowire]` quand possible.

## 9. Interface ambiguë (fondation)

Ajoute deux canaux au même contrat puis un consommateur qui demande un seul `ChannelInterface`, sans alias par défaut. Reproduis l'échec dans une configuration isolée pour ne pas bloquer les autres exercices.
- Attendu : message de compilation indiquant l'argument non résolu ; après choix explicite, le consommateur utilise uniquement SMS, tandis que l'itérateur global contient toujours les deux canaux.
- Preuve : diagnostic initial sauvegardé, `lint:container` réussi après correction, appel de service vérifiant le destinataire du message simulé.
- Indice : alias, `#[Target]` ou `#[Autowire(service: ...)]` ; un tag ne résout pas seul cette ambiguïté.

## 10. Ordre réel de décoration (application)

Décore SMS avec Retry priorité 20 et Logging priorité 0. Utilise une trace partagée et aucun envoi réel.
- Attendu : `log.before → retry.before → sms → retry.after → log.after` en succès. Inverser les priorités inverse l'imbrication. Sur échec SMS, la trace respecte ton contrat explicite de journalisation/relance.
- Preuve : même service demandé par son identifiant d'origine, ordre exact et exception conservée ; comparer la pile affichée en **8.1**.
- Indice : priorité haute appliquée en premier signifie plus proche du service d'origine.

## 11. Service privé optimisé (diagnostic)

Dans un `ContainerBuilder` isolé, enregistre un canal privé inutilisé puis compile ; dans une seconde expérience, référence-le depuis un consommateur public de test.
- Attendu : premier canal inaccessible après compilation ; seconde expérience, appel au consommateur réussi. Ne pas exiger que le canal interne possède encore un identifiant public récupérable : il peut être intégré au consommateur.
- Preuve : comparer définitions avant/après et résultat métier ; le container de test ne doit pas servir de justification à rendre toute l'application publique.
- Indice : `RemoveUnusedDefinitionsPass` et `InlineServiceDefinitionsPass` font des opérations distinctes.

## 12. Dépendance circulaire (approfondissement)

Reproduis `Notifier → Channel → Notifier` dans un container isolé. Propose ensuite une séparation de responsabilité, par exemple un journal commun sans dépendance vers le notificateur.
- Attendu : compilation initiale refusée avec chaîne du cycle ; version restructurée compilable et une seule notification/trace par appel. Deux notifications successives ne déclenchent aucune récursion.
- Indice : différer une dépendance n'est utile que si l'opération n'a pas besoin de la résoudre immédiatement.

## Défi final — notification explicable

Livre un `Notifier` avec canal par défaut, sélection explicite par clé, liste ordonnée de canaux et un canal décoré. Tous les canaux sont simulés. Attendu : canal inconnu refusé, zéro envoi externe, choix exact en environnement test, ordre vérifié et aucun cycle. Documente cinq diagnostics (alias, tag, arguments, décoration, compilation) et justifie chaque dépendance. Les tests doivent appeler le service, pas seulement inspecter les attributs.
