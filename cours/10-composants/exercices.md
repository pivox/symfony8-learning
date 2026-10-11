# Exercices — Module 10

[Parcours](../README.md) · [Cours](README.md) · [Approfondissement](approfondissement.md) · [Quiz](quiz.md)

Code dans `src/Components/` (namespace `App\Components`), tests dans `tests/Components/` (namespace `App\Tests\Components`) : `php bin/phpunit tests/Components`. Écris le test d'abord. Pour tout ce qui touche au disque, travaille dans un dossier temporaire (`sys_get_temp_dir()`) nettoyé en `tearDown()`.

## 1. Service dépendant du temps (Clock)
Crée `TokenIssuer` qui injecte `Symfony\Component\Clock\ClockInterface` et fournit `issue()` (token + date d'expiration à +15 min) et `isExpired($token)`.
- Tests avec `MockClock` : à t0 le token est valide ; après `sleep(900)` (ou `modify('+16 minutes')`) il est expiré — sans attente réelle.
- Question : pourquoi `new \DateTimeImmutable()` dans le service empêcherait-il de figer précisément le temps via `MockClock` ?
- Critère : `debug:autowiring clock` — quel service est injecté dans ton constructeur ?

## 2. Horloge globale et `ClockSensitiveTrait`
Écris `GreetingService::greet()` qui utilise `now()` (fonction globale) pour répondre « Bonjour » avant 12 h, « Bonsoir » sinon.
- Tests : `ClockSensitiveTrait` + `static::mockTime('2024-01-01 09:00')` puis `'2024-01-01 20:00'`.
- Réflexion : que se passe-t-il pour les tests suivants si tu appelles `Clock::set()` à la main sans restaurer ? Compare avec `ClockAwareTrait`.

## 3. PropertyAccess : lire/écrire
Crée des objets `Person` (propriétés privées + getters/setters, `isActive()`), `Address`, et un tableau de données.
- Tests : lecture `firstName`, `address.city`, `[contacts][0][email]` ; écriture via `setValue` ; `isReadable`/`isWritable` ; accès `address?.city` quand `address` est `null`.
- Cas d'erreur à tester : propriété inexistante (`NoSuchPropertyException`) ; mauvais type à l'écriture (`InvalidTypeException`/`UnexpectedTypeException`) — repère laquelle.
- Question : `__get` est actif par défaut : que change `disableMagicGet()` ? Pourquoi `__call` nécessite-t-il `enableMagicCall()` ?

## 4. PropertyAccess : collections (adder/remover)
Ajoute à une classe `Team` des `addMember()` / `removeMember()` et un `getMembers()`. Utilise `setValue($team, 'members', [...])`.
- Test : vérifie quels appels adder/remover ont lieu (compteur ou test double) selon la nouvelle liste.
- Réflexion : lien avec `CollectionType` et `by_reference: false` (module 09).

## 5. Finder : rapporteur de code
Écris `CodeStats::phpFiles(string $dir): array` qui retourne les chemins **relatifs** des `*.php` d'un dossier (profondeur ≤ 3, en excluant un sous-dossier `Fixtures`), triés par nom.
- Tests : crée une arborescence temporaire (fichiers cachés, `.git`, fichier `.txt`, dossier `Fixtures`) ; vérifie le résultat exact. Teste aussi un dossier inexistant (quelle exception ?).
- Question : `exclude('Fixtures')` ou `notPath('Fixtures')` — dans quels cas le résultat diffère-t-il ?

## 6. Filesystem : écriture sûre
Écris `ReportWriter` : crée l'arborescence cible, écrit un rapport avec `dumpFile`, puis le « publie » par `rename` vers un dossier final.
- Tests : le fichier existe, contenu exact, dossiers parents créés ; `rename` sans overwrite sur une cible existante ⇒ exception ; `mirror()` d'un dossier vers un autre ⇒ même contenu.
- Question : pourquoi `dumpFile` est-il préférable à `file_put_contents` pour un fichier lu par d'autres processus ?

## 7. Mime : construire un email

Préparation : suivre la [section Mime du cours](README.md) ; `Address` nécessite `egulias/email-validator`, absent du socle tant qu'il n'a pas été installé seul ou avec Mailer.
Écris `WelcomeEmailFactory::create(string $to): Email` : expéditeur nommé (`Address`), sujet, corps texte **et** HTML, une pièce jointe générée à la volée.
- Tests : en-têtes (`getFrom()`, `getTo()`, `getSubject()`), `getTextBody()` / `getHtmlBody()`, nombre de pièces jointes, type MIME du corps (`getBody()` est multipart de quel sous-type ?).
- Bonus : `MimeTypes::getDefault()->guessMimeType()` sur un petit fichier PNG/texte temporaire ; `getExtensions('image/png')`.
- Note : l'envoi réel nécessite `composer require symfony/mailer` (non demandé ici).

## 8. Process : lancer une commande
Écris `GitVersion::get()` (ou `php --version`) avec `Process` (tableau d'arguments).
- Tests : succès ⇒ sortie non vide ; commande inexistante ou code ≠ 0 ⇒ `ProcessFailedException` avec `mustRun()` ; `run()` seul ne lève pas (vérifie `isSuccessful()`) ; timeout très court sur `['sleep', '2']` ⇒ `ProcessTimedOutException`.
- Réflexion : tableau vs `Process::fromShellCommandline` — quand l'utiliser et comment passer une valeur sans risque d'injection ? Que renvoie `getExitCode()` avant la fin du process ?

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.

## Critères complémentaires pour les exercices 1–8

Appliquer la [méthode commune](../README.md). Les services intégrés se testent par `KernelTestCase` ; les composants autonomes peuvent aussi être explorés dans un petit test sans kernel.

| Exercice | Acceptation supplémentaire |
|---|---|
| 1 | Définir la borne : valide à 899 s, expiré à 900 s ; aucune attente réelle et fuseau explicite. |
| 2 | Tester 11:59:59 puis 12:00:00 dans le fuseau métier choisi ; pas de pollution de l'horloge globale après le test. |
| 3 | Null intermédiaire, propriété absente et mauvais type sont trois fixtures distinctes ; noter la classe d'exception exacte. |
| 4 | Membres A,B → B,C : retirer A et ajouter C exactement une fois, conserver B sans double appel. |
| 5 | Liste relative exacte et tri déterministe ; `in()` sur chemin absent → `DirectoryNotFoundException`, distincte d'une liste vide. |
| 6 | Contenu publié exact ; échec de publication sans écraser une cible existante ; nettoyage même si une assertion échoue. |
| 7 | Email texte+HTML+pièce jointe → corps externe `multipart/mixed`, partie alternative à l'intérieur ; aucun envoi réel. |
| 8 | Préférer `[PHP_BINARY, '-r', ...]` à `sleep`/Git pour les cas d'erreur portables ; `run()` peut lever une exception de timeout même s'il ne lève pas pour un simple code non nul. |

## 9. Parcours guidé — même instant, deux fuseaux

Écrire un service de salutation dépendant de Clock et d'un fuseau explicite. Fixer `2026-01-15 11:30:00 UTC` et comparer affichages UTC/Europe-Paris.

- Acceptation : même timestamp ; heure 11:30 UTC et 12:30 Paris ; salutation « Bonjour » dans le premier cas et « Bonsoir » dans le second selon le seuil midi.
- Ajouter un cas d'été (15 juillet) pour vérifier le décalage de Paris à +02:00 ; l'horloge originale conserve son fuseau.
- Indice : convertir pour l'affichage et les règles locales, pas pour changer l'instant enregistré.

## 10. Parcours intermédiaire — chemins et erreurs d'écriture

Un service lit une propriété imbriquée puis écrit un rapport dans un dossier temporaire. Définir explicitement si une valeur réellement `null` est admise.

- Acceptation : chemin valide → contenu attendu ; propriété mal orthographiée → erreur PropertyAccess, pas rapport vide ; parent de sortie remplacé par un fichier → exception Filesystem ; aucune publication finale sur échec.
- Tester séparément le chemin null-safe et une propriété inexistante. Nettoyer tous les fichiers après le test, succès comme échec.
- Indice : pour simuler une erreur disque, un fichier à la place d'un dossier est plus déterministe qu'un changement de permissions exécuté en root.

## 11. Parcours avancé — protocole d'un processus

Créer un petit service exécutant une commande PHP connue, avec timeout et résultat structuré (stdout, stderr, code) ou exception documentée.

- Acceptation : stdout `ok`, stderr `warning`, code 0 → succès ; stdout vide, stderr vide, code 7 → échec ; processus silencieux trop long → `ProcessTimedOutException`.
- Ajouter une valeur d'argument contenant espaces et caractères shell : elle doit être transmise comme un argument littéral, sans exécuter de commande ajoutée.
- Indice : tableau d'arguments et `PHP_BINARY` ; aucune assertion sur un délai exact à la milliseconde.

## 12. Parcours autonome — EventDispatcher hors HTTP

Créer un événement métier et trois listeners de priorités 20, 0 et -10 sur un `EventDispatcher` instancié à la main. Ajouter une variante stoppable.

- Acceptation : trace exacte 20 → 0 → -10 ; `dispatch()` retourne la même instance ; avec arrêt au listener 0, le listener -10 n'est pas appelé. Aucun kernel démarré.
- Ajouter un listener avec seulement `#[AsEventListener]`, sans l'enregistrer sur ce dispatcher : prouver qu'il ne s'exécute pas automatiquement.
- Indice : le dispatcher manipule des objets et des callables ; l'autoconfiguration appartient au conteneur, pas à l'attribut seul.

## Défi final — rapport CLI observable

Concevoir un service de rapport qui sélectionne les fichiers via Finder, date le résultat avec Clock, publie via Filesystem et déclenche un événement métier après réussite. Un Process optionnel vérifie un outil externe ; Mime peut préparer une notification sans l'envoyer.

Livrables : fixtures temporaires, tests d'un service appelé comme par une commande, trace d'événements et tableau des exceptions choisies. Acceptation : liste triée exacte, horodatage déterministe, aucune notification de succès après erreur d'écriture ou timeout, un seul événement après publication, dossier temporaire propre après test. Justifier les limites de l'atomicité d'un seul fichier et de la mémoire du processus. Indice final : isoler chaque frontière et prouver son échec avant d'assembler le service.

## 13. Distinguer recherche et instantané de noms — intermédiaire

Durée indicative : 35–50 min. Prérequis : exercices 5–6 et [parcours Finder](approfondissement.md).
Scénario : un service annonce les noms des documents `.txt` d'un dossier d'import temporaire.
Créer `a.txt` contenant `A`, `b.txt` contenant `B` et `ignore.csv` contenant `C` ; configurer le Finder avant d'ajouter `c.txt` contenant `C`.
Le service doit pouvoir produire un tableau de noms relatifs triés, sans conserver un Finder partagé entre ses appels.

- Réussite : première itération après création de `c.txt` → `['a.txt', 'b.txt', 'c.txt']` ; mémoriser ce tableau, puis créer `d.txt`.
- Réussite : le tableau mémorisé reste inchangé ; un nouvel appel réel au service retourne les quatre noms `.txt`, jamais `ignore.csv`.
- Cas négatif : racine inexistante → `DirectoryNotFoundException`, pas un tableau vide ; un fichier supprimé après relevé ne doit pas être présenté comme un contenu déjà capturé.
- Diagnostic : montrer qu'une seconde variable affectée au même Finder partage ses critères ; expliquer pourquoi le service reconstruit sa recherche.
- Preuve : `KernelTestCase` appelant le service, fixtures créées sous un dossier temporaire et nettoyage garanti après chaque cas.
- Indice 1 : configurer un itérateur et le matérialiser sont deux moments distincts.
- Indice 2 : comparer tableau de chaînes, tableau de `SplFileInfo` et nouvel appel à `getIterator()` ; seuls les noms déjà copiés sont figés ici.

## 14. Lire un protocole de sortie sans dépendre des fragments — avancé

Durée indicative : 55–75 min. Prérequis : exercice 11, buffers et [flux Process](approfondissement.md).
Scénario : un service lance un enfant PHP connu qui lit sur stdin les identifiants `R-1` et `R-2`, puis écrit une ligne JSON par identifiant sur stdout.
Fixer la sortie nominale à `{"id":"R-1","ok":true}\n{"id":"R-2","ok":true}\n`, stderr à `diagnostic\n` et code final à 0.
Passer les données à `setInput()` et utiliser `PHP_BINARY` avec un tableau d'arguments ; aucun shell ni serveur externe.
Le service ne publie sa liste qu'après fin réussie et validation de toutes les lignes.

- Réussite : appel réel au service → deux résultats dans l'ordre `R-1`, `R-2`, valeurs `ok` booléennes, diagnostic conservé séparément.
- Robustesse : l'enfant écrit successivement des demi-lignes ; le résultat est identique, sans imposer le nombre ni la taille des callbacks Process.
- Cas négatif A : dernière ligne tronquée, code 0 → erreur de protocole et aucune publication partielle.
- Cas négatif B : deux lignes valides, code 7 → échec d'exécution et aucune publication ; ne pas déduire le succès de la seule validité JSON.
- Preuve : tests de service avec trois programmes enfants déterministes, timeout court mais non chronométré et buffers réinitialisés entre appels.
- Indice 1 : un callback livre un fragment d'octets, pas nécessairement une ligne métier.
- Indice 2 : séparer buffer stdout incomplet, résultats provisoires, stderr et décision finale sur le code ; le reliquat final fait partie du protocole.
