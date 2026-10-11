# Exercices — Module 01

[Cours](README.md) · [Approfondissement](approfondissement.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Code dans `src/Php84/`, tests dans `tests/Php84/` (`php bin/phpunit tests/Php84`). Écris le test d'abord.

## 1. Property hooks
Crée `Temperature` avec une propriété `celsius` (float) et une propriété **virtuelle** `fahrenheit` lisible *et* modifiable (get + set convertissant vers `celsius`).
- Test : `fahrenheit = 212` ⇒ `celsius === 100.0`.
- Question : pourquoi `fahrenheit` n'a-t-elle pas de stockage ?

## 2. Hook `set` avec validation
Crée `Email` avec une propriété `value` dont le `set` normalise en minuscules et lève `InvalidArgumentException` si invalide (`filter_var`).
- Test : valeur invalide ⇒ exception ; `"Foo@Bar.com"` ⇒ `"foo@bar.com"`.

## 3. Asymmetric visibility
Crée `Counter` avec `public private(set) int $count` et une méthode `increment()`.
- Test : lecture OK ; écriture externe ⇒ `Error`. Que se passe-t-il avec `readonly` à la place ?

## 4. Interface avec hook
Déclare `interface HasName { public string $name { get; } }`. Implémente-la avec (a) une propriété stockée, (b) une propriété virtuelle.

## 5. `array_find*`
Écris `OrderService::firstExpensive(array $orders, float $min)` avec `array_find`, et `allPaid()` avec `array_all`.

## 6. Lazy ghost
À l'aide de `ReflectionClass::newLazyGhost`, crée un objet dont l'initialiseur incrémente un compteur. Prouve par test que l'initialiseur n'est appelé qu'**au premier accès** à une propriété.

## 7. PSR dans l'app (observation, pas de code)
Avec `debug:autowiring`, retrouve les alias PSR-3, PSR-11, PSR-14, PSR-20. Note pour chacun l'implémentation concrète.

> Indices progressifs : demande-moi « indice exo N » ; je te donne d'abord une piste, puis plus si besoin.


## Compléter les exercices 1–7 avant de poursuivre

Conserve tes classes et tests actuels ; cette liste précise les critères restants, sans fournir leur implémentation.

| Exercice | Cas d'acceptation complémentaires |
|---|---|
| 1 | Écrire `fahrenheit = 212` donne `celsius = 100.0` ; `32` donne `0.0` ; changer `celsius` modifie immédiatement la lecture Fahrenheit. Comparer les flottants avec une tolérance justifiée. |
| 2 | Normalisation à la construction **et** lors d'une réaffectation ; une valeur invalide après une valeur valide lève l'exception sans détruire la valeur précédente. |
| 3 | Deux incréments donnent 2 ; écriture externe refusée ; expliquer pourquoi `private(set)` n'est pas `readonly`, même si les deux refusent l'écriture externe. |
| 4 | Deux classes concrètes, une stockée et une virtuelle, satisfont `HasName`. Un mock de l'interface seul ne prouve pas cette conformité. |
| 5 | Premier élément strictement au-dessus du seuil ; égal au seuil exclu ; aucune correspondance et tableau vide donnent `null`. Ajouter `allPaid()` : tous payés → true, un impayé → false, vide → true. |
| 6 | Compteur 0 avant accès, 1 après lecture et encore 1 après une seconde lecture. Distinguer le test de réflexion PHP du test d'intégration du lazy service déjà présent. |
| 7 | Pour chaque PSR, noter contrat, alias réellement trouvé ou absence et composant fournisseur ; ne pas inventer Monolog ou un alias PSR-14. |

Indice : un test doit appeler la classe en cours d'exercice ; recalculer uniquement la même formule dans le test ne démontre pas que son setter fonctionne.

## 8. Typage et invariants (fondation)

Crée un value object `Quantity` immutable, entier strictement positif. Définis séparément le comportement d'un argument de mauvais type et d'une valeur entière hors domaine.
- Attendu : 1 accepté et lisible ; 0 et -1 refusés avec une exception métier ou `InvalidArgumentException` ; `'1'` provoque `TypeError` depuis un fichier appelant strict. La valeur ne peut pas être réaffectée.
- Preuve : tests séparés type/domaine/immutabilité ; expliquer l'effet de `declare(strict_types=1)` dans le fichier appelant.
- Indice : le type `int` ne porte pas la règle « positif ».

## 9. Enum et transitions (application)

Crée `OrderStatus` avec `draft`, `paid`, `cancelled` et une politique autorisant seulement `draft → paid` et `draft → cancelled`.
- Attendu : `tryFrom('unknown')` donne `null` ; `from('unknown')` lève `ValueError` ; les deux transitions autorisées passent, `paid → draft` et `cancelled → paid` sont refusées.
- Preuve : tableau de cas couvrant les 9 couples, y compris les transitions vers soi-même (refusées par ce contrat).
- Indice : distinguer conversion de chaîne et décision métier ; un `match` n'a pas besoin de cacher les cas manquants avec `default`.

## 10. Closures et gestion d'échec (diagnostic)

Construis un petit appelant recevant une `Closure` de traitement et une `Closure` de journalisation ; chaque exécution doit journaliser sa fin, même si le traitement échoue.
- Attendu : traitement réussi → résultat conservé et une trace ; traitement levant `RuntimeException` → même exception propagée et une trace ; avec `TypeError`, la trace existe aussi.
- Ajoute une expérience de capture : variable 2 puis 3, closure par valeur donnant 20 et par référence donnant 30 pour l'entrée 10.
- Indice : examiner `Throwable` et `finally` ; ne pas retourner depuis le bloc de nettoyage.

## 11. Traits et substituabilité (approfondissement)

Deux traits exposent `label()` avec des résultats différents. Crée une classe qui choisit explicitement une méthode et conserve l'autre sous un alias ; ajoute une interface commune à deux implémentations.
- Attendu : appel principal → `'court'`, alias → `'détaillé'` ; le même test de contrat accepte les deux implémentations. Une variante conflictuelle isolée échoue au chargement et ne casse pas la suite normale.
- Indice : `insteadof` sélectionne, `as` aliasse ; le trait ne remplace pas l'interface.

## Défi final — mini-domaine de commande

Compose quantité, statut et politique de transition dans un service sans base de données. Scénario : créer une commande, payer, refuser une annulation après paiement, puis chercher la première commande au-dessus d'un seuil avec le service existant. Attendu : état inchangé après échec, aucun résultat → `null`, trace d'échec unique et aucun effet partagé entre deux commandes. Livre les tests, les sorties PHPUnit et cinq lignes justifiant types, enum, exception et immutabilité. Pas de correction complète à produire.

## 12. Import paresseux avec arrêt anticipé

**Niveau : intermédiaire · Durée : 40–55 min · Prérequis : exercice 10 et [générateurs](approfondissement.md).**

Un écran ne veut afficher que les deux premiers codes d'un import potentiellement coûteux. Créer un service produisant un nouveau générateur à chaque appel et une trace indiquant chaque code effectivement produit.

Données : codes `A-10`, `B-20`, `C-30`, dans cet ordre. La variante d'échec remplace le troisième code par une donnée invalide qui doit lever `UnexpectedValueException` lorsqu'elle est atteinte.

Critères de réussite, par appels réels au service depuis les tests :

- Créer le générateur ne remplit pas la trace ; demander le premier élément ajoute seulement `A-10`.
- Consommer deux éléments puis arrêter produit `A-10`, `B-20`, sans production du troisième.
- Deux appels indépendants au service permettent deux parcours complets, avec les mêmes valeurs dans le même ordre.
- Le parcours complet de la variante invalide lève l'exception après les deux valeurs valides ; sa simple création ne la lève pas.
- Cas négatif : tenter un nouveau parcours du même générateur déjà terminé ne fournit pas magiquement trois nouveaux éléments. Relever l'échec PHP et le distinguer d'une nouvelle invocation du service.

Livrable : tests, trace de production et trois phrases expliquant à quel moment les calculs et l'erreur surviennent. Ne pas transformer immédiatement le générateur en tableau dans le scénario d'arrêt anticipé.

**Indice 1 :** distinguer « obtenir un objet Generator » et « demander une valeur à cet objet ».

**Indice 2 :** une méthode contenant `yield` reporte aussi l'exécution des instructions avant le premier `yield` ; si une validation doit être immédiate, réfléchir à la frontière entre méthode publique et producteur interne.

## 13. Brouillon indépendant et instantané immutable

**Niveau : intermédiaire · Durée : 45–60 min · Prérequis : exercice 8, `readonly`, section copie de l'[approfondissement](approfondissement.md).**

Un devis peut être dupliqué pour préparer une nouvelle offre. Créer un brouillon mutable contenant une date mutable et une ligne mutable, puis une opération explicite de duplication qui isole les données modifiables. Ajouter un instantané `readonly` représentant les valeurs confirmées, avec une date immutable.

Données : référence `D-07`, date `2026-02-10 12:00:00 UTC`, ligne `carnet`, quantité `2`, prix unitaire `350` centimes. Dans la copie seulement, passer à quantité `5` et avancer la date de deux jours.

Critères de réussite :

- Un appel à la duplication produit une instance distincte ; modifier sa ligne et sa date laisse l'original à quantité 2 et au 10 février.
- Deux duplications sont indépendantes entre elles ; l'instantané conserve un total de 700 centimes même si le brouillon évolue ensuite.
- L'instantané expose une `DateTimeImmutable` et aucune API ne permet de réaffecter sa quantité ; une tentative d'écriture externe échoue.
- Cas négatif : quantité 0 refusée, sans modifier la dernière valeur valide ni un instantané déjà créé.
- Une expérience séparée avec un simple clonage superficiel met en évidence le partage initial de l'objet ligne ; elle ne doit pas remplacer le test de l'opération finale.

Livrable : tests appelant les opérations du brouillon, avec assertions sur les valeurs après mutation et sur les identités utiles. Pas de sérialisation JSON comme raccourci de copie profonde.

**Indice 1 :** comparer uniquement les références des deux brouillons ne dit rien de leurs propriétés contenant d'autres objets.

**Indice 2 :** décider pour chaque donnée si elle doit être recopiée, partagée ou transformée en valeur immutable ; `__clone()` et un constructeur d'instantané répondent à des besoins différents.
