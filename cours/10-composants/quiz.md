# Quiz — Module 10 : Composants

Une ou plusieurs bonnes réponses : c'est indiqué. Réponses en bas.

**Q1.** (1 réponse) Que fait `MockClock::sleep(60)` ?
- A. Bloque le script 60 secondes
- B. Avance l'horloge simulée de 60 secondes sans attendre
- C. Lève une exception
- D. Ne fait rien

**Q2.** (1 réponse) Type retourné par `ClockInterface::now()` :
- A. `\DateTime`
- B. `int`
- C. `DatePoint`
- D. `\DateTimeInterface` non immuable

**Q3.** (2 réponses) Parmi ces éléments, lesquels existent dans le composant Clock ?
- A. `NativeClock`
- B. `MonotonicClock`
- C. `FrozenClock`
- D. `SystemClock`

**Q4.** (1 réponse) Quel trait fige le temps pour les tests via `mockTime()` ?
- A. `ClockAwareTrait`
- B. `ClockSensitiveTrait`
- C. `TimeMockTrait`
- D. `MockClockTrait`

**Q5.** (1 réponse) `PropertyAccessor::getValue($data, '[user][name]')` s'applique à :
- A. un objet avec propriété `user`
- B. un tableau (ou `ArrayAccess`)
- C. uniquement une Response
- D. une requête SQL

**Q6.** (3 réponses) Pour lire `$obj->firstName`, quels accès PropertyAccess essaie-t-il (configuration par défaut, méthodes existantes et propriété détectable via `__isset` pour l’accès magique) ?
- A. `getFirstName()`
- B. `isFirstName()`
- C. `__get('firstName')`
- D. `readFirstName()`

**Q7.** (1 réponse) Comment PropertyAccess modifie-t-il une collection d'objets ?
- A. Uniquement par `setX()`
- B. Par `addX()` / `removeX()` quand ils existent
- C. Par réflexion sur les propriétés privées
- D. Il ne gère pas les collections

**Q8.** (1 réponse) Que se passe-t-il si on itère un `Finder` sans avoir appelé `in()` (ni `append()`) ?
- A. Il cherche dans le dossier courant
- B. Une `LogicException` est levée
- C. Il retourne un résultat vide
- D. Il cherche dans tout le disque

**Q9.** (1 réponse) Par défaut, le Finder…
- A. inclut les fichiers cachés et `.git`
- B. ignore les fichiers cachés et les dossiers VCS
- C. ignore les fichiers PHP
- D. suit les liens symboliques

**Q10.** (1 réponse) `Finder::exclude('Fixtures')` :
- A. exclut les fichiers nommés `Fixtures`
- B. exclut des dossiers (relatifs au chemin de `in()`)
- C. exclut les fichiers contenant « Fixtures »
- D. est un alias de `notName`

**Q11.** (2 réponses) À propos de `Filesystem::dumpFile()` :
- A. L'écriture est atomique
- B. Il retourne `false` en cas d'erreur
- C. Il lève une `IOException` en cas d'erreur
- D. Il ajoute à la fin du fichier existant

**Q12.** (1 réponse) `Filesystem::remove()` sur un chemin inexistant :
- A. lève `FileNotFoundException`
- B. ne fait rien (pas d'erreur)
- C. retourne `false`
- D. crée le chemin

**Q13.** (1 réponse) Un `Email` avec `->text()` et `->html()` produit un corps :
- A. `text/plain` seul
- B. `multipart/alternative`
- C. `application/json`
- D. `multipart/signed`

**Q14.** (3 réponses) À propos de `Process` :
- A. `new Process(['ls', '-l'])` n'utilise pas de shell
- B. `run()` lève `ProcessFailedException` si le code de sortie ≠ 0
- C. `mustRun()` lève `ProcessFailedException` si le code de sortie ≠ 0
- D. Le timeout par défaut est de 60 secondes

**Q15.** (1 réponse) Pour exécuter `ls | wc -l` avec Process :
- A. `new Process(['ls | wc -l'])`
- B. `Process::fromShellCommandline('ls | wc -l')`
- C. `Process::pipe()`
- D. Impossible

---

## Questions de mise en situation — 16 à 20

**Q16.** (1 réponse) La même date est vue en UTC puis en Europe/Paris via `withTimeZone()`. Que conserve-t-elle ?

- A. Le timestamp du même instant
- B. Toujours la même heure affichée
- C. Un décalage Paris fixe toute l’année
- D. La mutation obligatoire de l’horloge originale

**Q17.** (1 réponse) `address?.city` résout quelle situation ?

- A. Toutes les fautes de frappe dans le chemin
- B. Un intermédiaire address existant mais null
- C. Une écriture arbitraire dans une propriété privée
- D. Un chemin de fichier hors du dossier autorisé

**Q18.** (1 réponse) Un processus écrit un avertissement sur stderr puis sort avec code 0. Quel diagnostic est exact ?

- A. mustRun() échoue forcément
- B. stderr doit être vide par contrat Symfony
- C. Le code indique un succès, stderr reste consultable
- D. getOutput() contient automatiquement stderr

**Q19.** (1 réponse) `run()` dépasse son timeout total. Que se passe-t-il ?

- A. Il ne peut jamais lever d’exception
- B. Il attend la fin sans limite
- C. Il lance automatiquement un nouveau processus
- D. Il lève `ProcessTimedOutException`

**Q20.** (1 réponse) Un listener porte `AsEventListener` mais aucun conteneur ni enregistrement manuel n’est utilisé avec un dispatcher autonome. Résultat ?

- A. Il est enregistré au premier dispatch
- B. Il n’est pas appelé par ce seul attribut
- C. Il est converti en message Messenger
- D. Il exige un événement kernel.request

## Réponses

1. **B.** `MockClock::sleep()` recalcule l'heure simulée (`MockClock.php`) ; `Clock`/`NativeClock` dorment vraiment.
2. **C.** `DatePoint` (sous-classe immuable de `\DateTimeImmutable`).
3. **A et B.** `NativeClock`, `MonotonicClock`, `MockClock`, `Clock` ; pas de `FrozenClock` ni `SystemClock`.
4. **B.** `Symfony\Component\Clock\Test\ClockSensitiveTrait` (`mockTime()`).
5. **B.** Les crochets désignent un index de tableau ; les points des propriétés d'objet.
6. **A, B et C.** Getters `get`, `is`, `has`, `can`, puis accès magique possible : `__get` est actif par défaut. `readFirstName()` n’est pas une convention reconnue. `__call` exige une activation explicite.
7. **B.** Adders/removers (utilisés par le `CollectionType` avec `by_reference: false`).
8. **B.** Il faut appeler `in()` ou `append()` avant d'itérer.
9. **B.** `ignoreDotFiles(true)` et `ignoreVCS(true)` par défaut.
10. **B.** `exclude()` cible des dossiers, relatifs aux dossiers de `in()`.
11. **A et C.** Écriture atomique (fichier temporaire + rename) ; exceptions plutôt que `false` ; pour ajouter : `appendToFile()`.
12. **B.** `remove()` est silencieux si le chemin n'existe pas.
13. **B.** Corps alternatif texte/HTML ; avec pièces jointes, le message devient `multipart/mixed` englobant.
14. **A, C et D.** Tableau = pas de shell ; `run()` ne lève pas, `mustRun()` oui ; timeout 60 s par défaut.
15. **B.** Les pipes nécessitent un shell : `fromShellCommandline` (jamais avec des entrées utilisateur concaténées ; utiliser `"${:VAR}"` + env).


## Réponses raisonnées — 16 à 20

16. **A.** Le fuseau modifie la représentation de l’instant. B confond instant et affichage ; C ignore l’heure d’été ; D contredit le retour d’une autre horloge.
17. **B.** Le null-safe gère une valeur intermédiaire nulle. A n’est pas sa fonction ; C dépend des accesseurs ; D relève d’un chemin Filesystem et d’une politique d’accès.
18. **C.** Les flux et le statut sont distincts. A et B inventent une règle d’échec ; D fusionne des sorties accessibles séparément.
19. **D.** L’absence de ProcessFailedException sur un code non nul ne retire pas les exceptions de timeout. A généralise abusivement ; B ignore la limite ; C invente un retry.
20. **B.** L’attribut est interprété par l’intégration du conteneur. A suppose une découverte automatique inexistante ; C et D confondent bus, événement métier et cycle HTTP.
