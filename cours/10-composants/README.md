# Module 10 — Composants : Clock, PropertyAccess, Finder, Filesystem, Mime, Process

## Objectifs, prérequis et repères

> Application : **Symfony 8.1.8, PHP ≥ 8.4**. La [certification Symfony](https://certification.symfony.com/exams/symfony.html) cible **8.0** : les mentions **8.1 uniquement** sont des compléments pour cette application, pas des acquis à attribuer à 8.0.

À la fin du module, tu dois rendre le temps déterministe, expliquer les chemins PropertyAccess, gérer les erreurs d'E/S et de processus, et déclencher un événement métier **sans HttpKernel**.

Prérequis : objets PHP, exceptions, services et tests ; [Console](../13-console/README.md) pour exposer un outil. Clock, PropertyAccess, Finder, Filesystem, Mime, Process et EventDispatcher sont disponibles (ce dernier comme dépendance). L'envoi d'email relève du [module Mailer](../19-mailer/README.md) ; Mime ne réalise pas le transport.

Applique la [méthode de travail commune](../README.md) : prédire, exécuter, observer, expliquer. Les exemples ci-dessous illustrent un mécanisme ; les exercices restent à réaliser. Le profiler n'est pas installé : utiliser les commandes et les tests proposés. Son installation facultative passerait par `composer require --dev symfony/profiler-pack`.

Tous ces composants sont **déjà installés** (voir `composer.json` : `symfony/clock`, `property-access`, `finder`, `filesystem`, `mime`, `process` en 8.1.*). Ce sont des composants **autonomes** : utilisables sans le framework, mais câblés par FrameworkBundle quand il y en a besoin.

## 1. Clock

Principe : ne **jamais** appeler `new \DateTimeImmutable()` / `time()` / `sleep()` dans du code métier testable ; injecter une horloge.

```php
use Symfony\Component\Clock\ClockInterface;

final class TokenFactory
{
    public function __construct(private ClockInterface $clock) {}

    public function expiresAt(): \DateTimeImmutable
    {
        return $this->clock->now()->modify('+1 hour');   // now() retourne un DatePoint
    }
}
```

Éléments (vérifiés dans `vendor/symfony/clock`) :
- `ClockInterface` (étend `Psr\Clock\ClockInterface`, PSR-20) : `now(): DatePoint`, `sleep(float|int $seconds)`, `withTimeZone()`.
- Implémentations : `NativeClock` (horloge réelle), `MonotonicClock` (pour mesurer des durées), `MockClock` (tests), `Clock` (**horloge globale** : `Clock::get()`, `Clock::set()`).
- **`DatePoint`** : sous-classe immuable de `\DateTimeImmutable` (6.4), avec exceptions strictes ; `setMicrosecond()/getMicrosecond()` (7.1).
- Fonction globale `now()` (6.3) : `now()` (horloge globale) ou `now('+1 day')`.
- `ClockAwareTrait` (6.3) : fournit `$this->now()` ; l'horloge est injectée par `#[Required] setClock()` (sinon `new Clock()` global).
- `MockClock` :
```php
$clock = new MockClock('2024-01-15 10:00:00');
$clock->sleep(3600);              // ne dort PAS : avance simplement l'horloge de 1 h
$clock->modify('+1 day');
$clock->now()->format('c');
```
- Dans un `TestCase` : `use ClockSensitiveTrait;` puis `static::mockTime('2024-01-01')` (remplace l'horloge globale par un `MockClock`, restaurée avant chaque test).
- Dans le conteneur : service `clock` (classe `Symfony\Component\Clock\Clock`, privé), alias autowirables `Symfony\Component\Clock\ClockInterface` et `Psr\Clock\ClockInterface` (vérifié avec `debug:autowiring clock`). `Clock::set()` agit sur l'horloge **globale** que ce service utilise.
- Injection d'une horloge particulière : `#[Autowire(service: 'clock')]` ou `#[Target]`.

### Piège
- `sleep()` de `Clock`/`NativeClock` dort réellement ; celui de `MockClock` **avance le temps**, sans attente.
- `now()` retourne un `DatePoint` (immuable) : `modify()` retourne un **nouvel** objet.
- Appeler `new \DateTimeImmutable()` directement contourne l'horloge : le test ne peut plus figer le temps.

## 2. PropertyAccess

```php
$accessor = PropertyAccess::createPropertyAccessor();   // ou createPropertyAccessorBuilder()->...->getPropertyAccessor()

$accessor->getValue($person, 'firstName');            // objet : getFirstName(), isFirstName(), hasFirstName(), canFirstName() ou propriété publique
$accessor->setValue($person, 'address.city', 'Paris'); // chemin pointé (objets)
$accessor->getValue($array, '[user][name]');           // tableaux : crochets
$accessor->getValue($person, 'address?.city');         // null-safe : retourne null si address est null
$accessor->isReadable($person, 'x'); $accessor->isWritable($person, 'x');
```
- Lecture objet (vérifié : `ReflectionExtractor::$defaultAccessorPrefixes`) : préfixes `get`, `is`, `has`, `can`, puis propriété publique ; `__get`/`__set` **activés** par défaut ; `__call` **désactivé** par défaut (`enableMagicGet()`, `enableMagicCall()`, `enableMagicMethods()` sur le builder). Pour un objet à propriétés virtuelles, définir aussi `__isset()` afin que la propriété soit détectable ; `__get()` seul ne garantit pas la lecture.
- Écriture : `setX()`, propriété publique, et pour les **collections** `addX()` / `removeX()` (adder/remover) — c'est ce qui sert au `CollectionType` (module 09).
- Notation : `.` pour les propriétés, `[x]` pour les index de tableau / `ArrayAccess`. Un chemin peut mélanger : `children[0].name`.
- Exceptions : `NoSuchPropertyException` (propriété inexistante), `NoSuchIndexException` (index absent d'un tableau, si `enableExceptionOnInvalidIndex()`), `UnexpectedTypeException`, `InvalidTypeException` (mauvais type à l'écriture), `UninitializedPropertyException` (propriété typée non initialisée).
- Avec `disableExceptionOnInvalidPropertyPath()`, un chemin invalide retourne `null`.
- 8.1 : `PropertyPath::append($base, $sub)` pour joindre des fragments de chemin.
- Utilisé par : Form, Serializer (`ObjectNormalizer`), Validator (`Expression`, `Valid`), SecurityBundle.

## 3. Finder

Recherche de fichiers/dossiers, **itérable** de `Symfony\Component\Finder\SplFileInfo` (enrichit `\SplFileInfo` avec `getRelativePath()`, `getRelativePathname()`, `getContents()`).

```php
$finder = Finder::create()
    ->files()
    ->in([__DIR__.'/src', __DIR__.'/tests'])
    ->name('*.php')                        // glob, regex ('/Test\.php$/') ou tableau
    ->notName('*Test.php')
    ->path('Controller')                   // filtre sur le chemin
    ->exclude('Fixtures')                  // exclut des DOSSIERS (relatifs à in())
    ->depth('< 2')                         // comparateurs : '== 0', '> 1'…
    ->size('>= 1K')                        // K, M, G, Ki, Mi…
    ->date('since yesterday')
    ->contains('namespace App')
    ->sortByName();

foreach ($finder as $file) { echo $file->getRelativePathname(); }
$finder->hasResults(); count($finder);
```
- `in()` est **obligatoire** avant d'itérer (sinon `LogicException`) et lève `DirectoryNotFoundException` si le dossier n'existe pas ; plusieurs dossiers : tableau.
- Par défaut le Finder **ignore** les fichiers cachés (`.`) et les dossiers VCS (`.git`…) : `ignoreDotFiles(false)`, `ignoreVCS(false)`, `ignoreVCSIgnored(true)` (respecte `.gitignore`).
- `append()` ajoute un autre itérateur ; `filter(\Closure, prune)` pour un filtre perso ; `followLinks()` pour suivre les liens symboliques ; `ignoreUnreadableDirs()`.
- Tris : `sortByName()`, `sortBySize()`, `sortByType()`, `sortByModifiedTime()`, `sortByExtension()`, `sort(\Closure)`, `reverseSorting()`.
- 8.1 : `useUnixPaths()`. 6.4 : élagage anticipé de dossiers avec `filter(..., prune: true)`.
- Le Finder est **paresseux** : rien n'est lu avant l'itération/`count()`/`hasResults()`.

## 4. Filesystem

Opérations de fichiers sûres, avec **exceptions** (`IOException`, `FileNotFoundException`) au lieu de `false`.

```php
$fs = new Filesystem();
$fs->mkdir('/tmp/a/b/c');                   // récursif
$fs->dumpFile('/tmp/a/f.txt', 'x');         // écriture ATOMIQUE (fichier temporaire + rename)
$fs->appendToFile('/tmp/a/f.txt', 'y');
$fs->readFile('/tmp/a/f.txt');              // 7.1
$fs->copy($src, $dst);                      // n'écrase que si $src plus récent (sauf $overwriteNewerFiles)
$fs->rename($old, $new, overwrite: false);
$fs->remove(['/tmp/a']);                    // récursif, silencieux si absent
$fs->exists('/tmp/a');                      // un seul chemin ou iterable : tous doivent exister
$fs->mirror($srcDir, $dstDir);              // synchronise ; options (delete, override, follow_symlinks…)
$fs->makePathRelative($end, $start);  $fs->isAbsolutePath($p);
$fs->tempnam('/tmp', 'prefix_');  $fs->symlink(...);  $fs->chmod(...)
```
- `dumpFile` est atomique : lecteurs concurrents ne voient jamais un fichier à moitié écrit.
- 8.1 : l'option `copy_on_windows` de `mirror()` est **dépréciée** (utiliser `follow_symlinks`).
- Les tests de Filesystem utilisent `sys_get_temp_dir()` + `tempnam` ; nettoyer dans `tearDown()`.
- `Path` (utilitaire, classe `Symfony\Component\Filesystem\Path`) pour manipuler les chemins (`canonicalize`, `join`, `makeRelative`) — à tester dans l'app avant de l'utiliser.

## 5. Mime

> `symfony/mime` sert aussi de base à Mailer (`composer require symfony/mailer` pour envoyer).

```php
$email = (new Email())
    ->from(new Address('no-reply@example.com', 'Acme'))
    ->to('user@example.com')->cc('cc@example.com')
    ->subject('Bienvenue')
    ->text('Bonjour')                               // corps texte
    ->html('<p>Bonjour</p>')                        // corps HTML
    ->attachFromPath('/tmp/doc.pdf')                // pièce jointe
    ->embedFromPath('/tmp/logo.png', 'logo');       // image inline (cid:logo)
```
- `Email` produit un message **multipart** (`multipart/alternative` si text+html, `multipart/mixed` avec pièces jointes, `multipart/related` pour embed) ; `getBody()` retourne la partie (`AbstractPart`).
- `Address` : `new Address('a@b.c', 'Nom')`, `Address::create('Nom <a@b.c>')`.
- `Message` / `RawMessage` / `DraftEmail` ; `DataPart` (pièce jointe ; `DataPart::fromPath`), `TextPart`.
- Si Twig est présent : `TemplatedEmail` (twig-bridge) rend `htmlTemplate()`/`textTemplate()`.
- **Types MIME** : `MimeTypes::getDefault()->guessMimeType($path)` (nécessite `ext-fileinfo` ou la commande `file`), `getExtensions('image/jpeg')`, `getMimeTypes('jpg')`. `registerGuesser()` pour un guesser perso.
- 8.0 : `__sleep/__wakeup` remplacés par `__serialize/__unserialize` sur les `AbstractPart` (impact si tu sérialises un email, ex. via Messenger).

## 6. Process

```php
$process = new Process(['git', 'status', '--short'], cwd: '/repo', timeout: 10);   // commande = TABLEAU d'arguments
$process->run();                                  // synchrone, retourne le code de sortie
if (!$process->isSuccessful()) { throw new ProcessFailedException($process); }
echo $process->getOutput();    echo $process->getErrorOutput();

$process->mustRun();                              // lève ProcessFailedException si code != 0
```
- **Tableau d'arguments** (pas de shell, échappement automatique : sûr). `Process::fromShellCommandline('ls | wc -l')` pour du shell (pipes…) : variables passées via `$process->run(null, ['NAME' => $v])` et référencées `"${:NAME}"` — **ne jamais** concaténer une entrée utilisateur.
- **Timeout** : `timeout` par défaut **60 s** (`setTimeout(null)` pour désactiver), `idleTimeout` pour le silence ; dépassement ⇒ `ProcessTimedOutException`. Pour un process asynchrone, appelle `checkTimeout()` toi-même dans ta boucle.
- Asynchrone : `start()`, `isRunning()`, `wait()`, `waitUntil(callable)`, `getIncrementalOutput()`, `stop(timeout, signal)`, `signal()`; itération en flux : `foreach ($process as $type => $data)` (`getIterator`).
- Entrée : `setInput()`, `InputStream` pour alimenter en continu. Env : `setEnv()` / argument `$env` (variables d'`$_ENV` héritées par défaut).
- Exceptions : `ProcessFailedException` (code ≠ 0 via `mustRun`), `ProcessTimedOutException`, `ProcessSignaledException`, `ProcessStartFailedException`, `RuntimeException`, `LogicException`.
- `PhpSubprocess` / `PhpProcess` / `PhpExecutableFinder` / `ExecutableFinder` : lancer du PHP avec la config du parent ; trouver un binaire.
- `getExitCode()` est `null` tant que le process n'est pas terminé ; `isSuccessful()` = code 0.
- Messenger : `RunProcessMessage` / `RunProcessMessageHandler` (6.4) permettent de lancer un process via un bus ; `RunProcessMessage::fromShellCommandline()` (7.3).
- 7.1 : `setIgnoredSignals()` pour ne pas propager certains signaux.
- `tty` / `pty` : `setTty(true)` pour les commandes interactives (`isTtySupported()`).

## Ce qui change vs Symfony 6/7
- Clock : composant stable depuis 6.2 ; `DatePoint` (6.4), `now()` (6.3), `getMicrosecond` (7.1).  Pas d'entrée 8.0/8.1 dans le `CHANGELOG` du composant.
- PropertyAccess : `PropertyPathInterface::isNullSafe()` (7.0) ; `setCacheItemPool()` exige un argument explicite (7.0) ; 8.1 `PropertyPath::append()`.
- Finder : 8.1 `useUnixPaths()` ; 6.2 `sortByExtension/Size/CaseInsensitiveName`.
- Filesystem : 7.1 `readFile()` ; 8.1 `mirror()` option `copy_on_windows` dépréciée.
- Mime : 8.0 `__serialize/__unserialize` sur les parts.
- Process : 7.1 `setIgnoredSignals()` ; 7.3 `RunProcessMessage::fromShellCommandline()`.
- (Détails de dépréciations/suppressions : lire le `CHANGELOG.md` de chaque composant dans `vendor/symfony/<composant>/`.)

## Piège certification
- `MockClock::sleep()` n'attend pas ; `Clock::get()` est global (état partagé entre tests si non restauré — d'où `ClockSensitiveTrait`).
- `Process` : commande **tableau** vs **chaîne shell** ; timeout 60 s par défaut ; `run()` ne lève pas pour un code de sortie non nul (`mustRun()` oui), mais peut lever une exception de timeout ; `getOutput()` après `start()` sans `wait()` = sortie partielle.
- `Filesystem::dumpFile()` remplace un fichier atomiquement, sans rendre une suite d’opérations transactionnelle ; `copy()` n'écrase pas un fichier cible plus récent par défaut.
- `Finder::in()` obligatoire ; ignore dotfiles et VCS par défaut ; `exclude()` s'applique aux **dossiers** (pour des fichiers : `notName`/`notPath`).
- PropertyAccess : `[index]` (tableau) vs `.prop` (objet) ; `__get`/`__set` activés, `__call` désactivé par défaut.
- Mime : `text()` + `html()` ⇒ `multipart/alternative` ; l'envoi réel est le travail de Mailer (non installé).

## Observer dans l'app
- `bin/console debug:autowiring clock` et `debug:container clock`.
- `bin/console debug:container --tag=...` ; `debug:autowiring Filesystem` (service `filesystem` autowirable ; Finder et Process s'instancient avec `new`).
- `bin/console list` ; écrire une commande de test qui utilise Finder/Process : `bin/console make:command` (maker-bundle installé).
- Tests : `php bin/phpunit tests/Components` ; `ClockSensitiveTrait::mockTime()`.
- Après installation facultative du profiler : profiler : le panneau **Time** (Stopwatch) montre les durées, utile avec `MonotonicClock`.

## Doc officielle
- https://symfony.com/doc/current/components/clock.html
- https://symfony.com/doc/current/components/property_access.html
- https://symfony.com/doc/current/components/finder.html
- https://symfony.com/doc/current/components/filesystem.html
- https://symfony.com/doc/current/components/mime.html
- https://symfony.com/doc/current/components/process.html

## 7. Frontières : heure, chemins, disque et processus

### Un instant n'est pas son affichage local

```php
use Symfony\Component\Clock\MockClock;

$utc = new MockClock('2026-01-15 10:00:00 UTC');
$paris = $utc->withTimeZone('Europe/Paris');
// now()->format('H:i') : 10:00 pour UTC, 11:00 pour Paris.
// now()->getTimestamp() : identique pour les deux.
```

`withTimeZone()` retourne une autre horloge, sans modifier la première. Pour décider d'une expiration, comparer des instants ; pour « Bonjour avant midi », définir le fuseau métier puis lire l'heure locale. Ne pas figer un décalage `+01:00` si le besoin est « Europe/Paris » toute l'année. Un jour civil autour d'un changement d'heure ne vaut pas toujours 86 400 secondes : écrire deux tests distincts si le contrat distingue « demain à la même heure » et « dans 24 heures ».

### Un chemin invalide n'est pas une donnée vide

Avec PropertyAccess, `address?.city` permet un intermédiaire **existant et null**. Ce n'est pas un joker qui ignore n'importe quelle faute dans `adress.city`. Distinguer propriété inexistante, propriété typée non initialisée, valeur `null` et type incorrect à l'écriture. `isReadable()`/`isWritable()` permettent de sonder l'accès ; avaler toutes les exceptions en retournant `null` efface la distinction entre un bug de chemin et une valeur réellement absente. Sources : `PropertyAccessorBuilder.php` et [PropertyAccess 8.0](https://symfony.com/doc/8.0/components/property_access.html).

Un chemin Filesystem est une autre notion : `Path::canonicalize()` simplifie les segments, mais n'autorise pas un utilisateur à sortir d'un dossier de travail et ne résout pas à lui seul les liens symboliques. Pour les exercices, choisir un dossier temporaire contrôlé. Un test d'erreur portable peut créer un **fichier** à l'endroit où un répertoire parent est attendu, puis tenter un `dumpFile()` en dessous : cela échoue même lorsque les tests s'exécutent avec des privilèges élevés. Un simple `chmod` n'est pas fiable dans ce dernier cas.

L'atomicité de `dumpFile()` concerne le remplacement d'un fichier. Elle ne rend pas atomique une suite « lire compteur → incrémenter → écrire » et ne garantit pas qu'un rapport réparti dans trois fichiers sera publié tout entier. Si un véritable verrou devient nécessaire dans un prolongement, choisir `symfony/lock` ; ne pas simuler l'exclusion mutuelle avec un drapeau fichier. Ce prolongement sort du socle retenu ici.

### stdout, stderr et code de sortie sont trois informations différentes

```php
use Symfony\Component\Process\Process;

$p = new Process([PHP_BINARY, '-r',
    'fwrite(STDOUT, "rapport"); fwrite(STDERR, "avertissement"); exit(0);',
]);
$p->run();
// getOutput() === "rapport", getErrorOutput() === "avertissement".
// isSuccessful() === true : écrire sur stderr ne force pas un échec.
```

Inversement, un code non nul peut arriver sans aucun texte sur stderr. `run()` permet de lire le code ; `mustRun()` transforme un code non nul en `ProcessFailedException`. Les deux peuvent lever une `ProcessTimedOutException` : « run ne lève pas pour un code non nul » ne veut pas dire « run ne lève jamais ». Le timeout total limite la durée de vie ; l'idle timeout limite le silence observé sur stdout/stderr et suppose que la sortie soit activée. Utiliser `[PHP_BINARY, '-r', 'usleep(200000);']` évite de dépendre d'un exécutable Unix `sleep` dans les tests. Pour des durées, vérifier l'exception, pas un chronométrage à la milliseconde.

Observer : assertions sur les trois sorties, `debug:autowiring Clock`, test des chemins exceptionnels. [Process 8.0](https://symfony.com/doc/8.0/components/process.html) et `vendor/symfony/process/Process.php` précisent les limites.

## 8. EventDispatcher sans Kernel

EventDispatcher ne dépend pas de HTTP. Une classe événement métier peut vivre dans un script CLI ou un service autonome. Sans conteneur, l'attribut `AsEventListener` n'enregistre rien tout seul : enregistrer explicitement un callable ou un subscriber sur le dispatcher.

```php
use Symfony\Component\EventDispatcher\EventDispatcher;

final readonly class ReportReady
{
    public function __construct(public string $name) {}
}

$dispatcher = new EventDispatcher();
$trace = [];
$dispatcher->addListener(ReportReady::class,
    function (ReportReady $event) use (&$trace): void {
        $trace[] = $event->name;
    },
);
$event = new ReportReady('octobre');
$returned = $dispatcher->dispatch($event);
// $trace === ['octobre'] ; $returned === $event.
```

Sans nom explicite, le nom est la classe de l'objet ; un nom explicite sélectionne ses propres listeners. Une priorité plus élevée s'exécute d'abord. Le dispatch est synchrone : les listeners ont terminé lorsque `dispatch()` rend la main, sauf s'ils lancent eux-mêmes du travail différé. Pour permettre `stopPropagation()`, utiliser un événement implémentant `StoppableEventInterface`, par exemple en étendant `Symfony\Contracts\EventDispatcher\Event` ; la simple présence d'un objet readonly ne donne pas cette fonctionnalité.

Ce mécanisme et Messenger répondent à des besoins différents : EventDispatcher notifie des listeners immédiatement ; Messenger fournit bus, middleware et transports. Tester l'ordre dans un tableau de trace sans démarrer le kernel prouve que l'événement métier n'est pas un événement `kernel.*`. Source locale : `event-dispatcher/EventDispatcher.php`.
