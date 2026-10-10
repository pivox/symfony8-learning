# Quiz — Module 01 : PHP 8.4 & PSR

Réponds sans regarder le README. Sauf mention contraire, **une seule** bonne réponse. Les questions marquées *(plusieurs réponses)* ont 2 bonnes réponses ou plus. Réponses et explications en bas.

---

**Q1.** Quelle est la version minimale de PHP exigée par Symfony 8 ?
- A. 8.2
- B. 8.3
- C. 8.4
- D. 8.5

**Q2.** Que se passe-t-il avec ce code ?
```php
class P { public string $label { get => 'x'; } }
$p = new P();
$p->label = 'y';
```
- A. `label` devient `'y'`
- B. Une `Error` est levée : propriété virtuelle en lecture seule
- C. Une `TypeError` est levée
- D. Rien, l'écriture est ignorée silencieusement

**Q3.** Parmi ces propositions, lesquelles décrivent correctement une propriété *virtuelle* ? *(plusieurs réponses)*
- A. Aucun hook ne lit ni n'écrit `$this->prop`
- B. Elle ne consomme pas de stockage dans l'objet
- C. Elle doit obligatoirement être `readonly`
- D. Elle peut avoir à la fois un hook `get` et un hook `set`

**Q4.** Quelle déclaration est **invalide** ?
- A. `public private(set) int $n = 0;`
- B. `public protected(set) string $s = '';`
- C. `protected public(set) int $n = 0;`
- D. `private(set) int $n = 0;`

---

**Q5.** Quelle est la différence principale entre `public private(set) int $x` et `public readonly int $x` ?
- A. Aucune
- B. `private(set)` permet à la classe de modifier la valeur plusieurs fois, `readonly` n'autorise qu'une seule initialisation
- C. `readonly` permet la modification depuis l'extérieur
- D. `private(set)` n'est compatible qu'avec les propriétés non typées

**Q6.** Que fait `new Foo()->bar()` en PHP 8.4 ?
- A. Erreur de syntaxe
- B. Équivalent à `(new Foo())->bar()`
- C. Équivalent à `new (Foo()->bar())`
- D. Appelle la méthode statique `bar`

**Q7.** Que retourne `array_find([1, 2, 3, 4], fn($v) => $v > 2)` ?
- A. `[3, 4]`
- B. `2` (clé)
- C. `3`
- D. `true`

**Q8.** Quelles fonctions sont nouvelles en PHP 8.4 ? *(plusieurs réponses)*
- A. `array_find`
- B. `array_find_key`
- C. `array_any`
- D. `array_filter`
- E. `array_all`

**Q9.** Quelle API PHP 8.4 permet de créer un objet dont l'état n'est initialisé qu'au premier accès ?
- A. `ReflectionClass::newInstanceWithoutConstructor()` uniquement
- B. `ReflectionClass::newLazyGhost()` / `newLazyProxy()`
- C. `WeakReference::create()`
- D. `__lazy()`

**Q10.** Une interface peut-elle déclarer une propriété ?
- A. Non, jamais
- B. Oui, via un hook : `public string $name { get; }`
- C. Oui, mais uniquement `static`
- D. Oui, mais seulement avec un stockage obligatoire

**Q11.** Quelle PSR décrit l'autoloading ?
- A. PSR-1
- B. PSR-3
- C. PSR-4
- D. PSR-11

**Q12.** Quelles méthodes définit `Psr\Container\ContainerInterface` (PSR-11) ? *(plusieurs réponses)*
- A. `get(string $id)`
- B. `has(string $id)`
- C. `set(string $id, mixed $service)`
- D. `remove(string $id)`

**Q13.** Quelle PSR correspond à l'horloge ?
- A. PSR-14
- B. PSR-18
- C. PSR-20
- D. PSR-6

**Q14.** Pour un client HTTP au sens PSR, quelle PSR est concernée ?
- A. PSR-7
- B. PSR-15
- C. PSR-17
- D. PSR-18

**Q15.** Quelle affirmation est vraie à propos de PSR-15 dans Symfony ?
- A. `HttpKernel` implémente nativement `RequestHandlerInterface`
- B. HttpKernel n’implémente pas PSR-15 ; une adaptation est nécessaire, avec conversion des messages PSR-7 si besoin
- C. Elle remplace `EventDispatcherInterface`
- D. Elle définit le logger

---


## Questions complémentaires — mise en situation

Une seule bonne réponse par question.

**Q16.** Une backed enum ne contient pas la valeur `missing`. Quel appel retourne `null` ?
- A. `Status::from('missing')`
- B. `Status::tryFrom('missing')`
- C. `Status::cases()`
- D. `new Status('missing')`

**Q17.** Un fichier avec `strict_types=1` appelle `f(int $n)` avec `'7'`. Que se passe-t-il ?
- A. Conversion systématique en 7
- B. Retour null
- C. TypeError
- D. Cela dépend seulement du fichier qui déclare f

**Q18.** Une propriété readonly contient un objet mutable. Quelle affirmation est correcte ?
- A. L’objet contenu devient immutable
- B. La propriété ne peut plus être réaffectée après initialisation, mais l’objet peut muter
- C. Toute lecture est interdite
- D. Readonly valide les invariants métier

**Q19.** Une arrow function capture `$n = 2`, puis le code affecte `$n = 3`. Que lit la fonction ?
- A. 2
- B. 3
- C. null
- D. Une référence obligatoire

**Q20.** Quel catch intercepte à la fois RuntimeException et TypeError ?
- A. `catch (Exception $e)`
- B. `catch (Error $e)`
- C. `catch (Throwable $e)`
- D. Aucun

## Réponses

**Q1 — C.** Symfony 8 exige PHP >= 8.4 (voir `composer.json`).

**Q2 — B.** Une propriété avec seulement un hook `get` et sans stockage est virtuelle donc en lecture seule ; l'écriture lève une `Error`.

**Q3 — A, B, D.** Virtuelle = aucun stockage (aucun hook ne référence `$this->prop`). Elle peut avoir `get` et `set` (ex. `fahrenheit`). Elle ne peut pas être `readonly` (C faux).

**Q4 — C.** La visibilité du `set` doit être égale ou plus restrictive que celle du `get` : ici `get` est `protected` et `set` serait `public`, interdit. A et B sont valides ; D l'est aussi (la visibilité de lecture est alors `public` par défaut).

**Q5 — B.** `readonly` interdit toute réécriture après initialisation ; `private(set)` laisse la classe modifier la propriété. Les deux exigent une propriété typée.

**Q6 — B.** Depuis 8.4, les parenthèses autour de `new` ne sont plus nécessaires pour chaîner un appel.

**Q7 — C.** `array_find` retourne la **valeur** du premier élément qui satisfait le callback (`3`). `array_find_key` retournerait la clé (`2`).

**Q8 — A, B, C, E.** `array_filter` existe depuis PHP 4/5.

**Q9 — B.** `newLazyGhost()` et `newLazyProxy()` (PHP 8.4). Symfony les utilise pour les lazy services (module 03).

**Q10 — B.** L'interface impose une capacité de lecture (ou d'écriture avec `set;`), pas un stockage.

**Q11 — C.** PSR-4 (`autoload.psr-4` dans `composer.json`). PSR-3 = logger, PSR-11 = container.

**Q12 — A, B.** PSR-11 ne contient que `get` et `has` ; il n'y a pas de `set` dans la norme.

**Q13 — C.** PSR-20 : `Psr\Clock\ClockInterface`. Le composant Clock de Symfony en fournit une implémentation (`Symfony\Component\Clock\ClockInterface` étend celle de PSR).

**Q14 — D.** PSR-18 = client HTTP ; PSR-7 = messages ; PSR-17 = factories ; PSR-15 = handlers/middleware serveur.

**Q15 — B.** HttpKernel travaille avec `Request`/`Response` de HttpFoundation, pas avec PSR-7/15 ; le bridge convertit les messages, mais ne suffit pas à lui seul à implémenter le contrat de handler PSR-15.

**Q16 — B.** `tryFrom` exprime une conversion pouvant échouer sans exception. A lève `ValueError`, C liste les cas sans convertir, D est interdit : une enum ne s’instancie pas avec `new`.

**Q17 — C.** Le mode strict du fichier appelant gouverne ce paramètre scalaire. A correspond à une coercition non stricte, B n’est pas un mécanisme de typage, D inverse le rôle appelant/déclaration.

**Q18 — B.** Readonly porte sur la propriété. A confond immutabilité superficielle et profonde, C contredit une propriété lisible, D demande une validation métier indépendante.

**Q19 — A.** La capture implicite se fait par valeur. B et D conviendraient à une capture explicite par référence dans une closure classique ; C n’a aucune raison de remplacer la valeur capturée.

**Q20 — C.** Les deux familles implémentent Throwable. A manque TypeError, B manque RuntimeException, D ignore leur interface commune ; intercepter ne signifie pas devoir masquer l’erreur.

