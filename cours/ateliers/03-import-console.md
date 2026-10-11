# Atelier 03 — Un import Console contrôlé et reproductible

[Ateliers](README.md) · [Composants](../10-composants/README.md) · [Console](../13-console/README.md) · [Validator](../12-validator/README.md)

**Niveau :** intermédiaire à avancé. **Durée :** 3–5 h. **Prérequis :** modules 01, 03, 10, 12, 13 et 14. Code dans `src/Ateliers/Import/`, tests dans `tests/Ateliers/Import/`.

## Mission et données

Crée `app:training:import INPUT --output=OUTPUT [--dry-run]`. Elle lit un fichier CSV local, vérifie toutes ses lignes, puis écrit un rapport JSON seulement si le lot entier est valide. Aucun appel externe, aucune base et aucun envoi de messages.

Le CSV utilise la virgule, les guillemets doubles et l’en-tête exact `reference,title,seats`. Pour un premier lot valide :

```csv
reference,title,seats
A-01,HTTP,12
A-02,Symfony,8
A-03,"Tests, diagnostic",4
```

La référence suit `[A-Z]-[0-9]{2}` et doit être unique dans le fichier. Le titre n’est pas vide après trim. `seats` représente un entier décimal de 1 à 30, sans signe ni partie décimale. Refuser `3x`, `2.5` et `-1` avant toute conversion qui les masquerait. Une ligne physiquement vide est ignorée mais conserve son effet sur les numéros de ligne suivants. Le fichier d’en-tête seul constitue un lot invalide.

Avec PHP 8.4, vérifie la signature du lecteur CSV et passe explicitement le paramètre d’échappement voulu ; l’atelier utilise la chaîne vide. Le titre contenant une virgule doit rester une seule colonne.

## Jalon 1 — Analyse et violations

Sépare la lecture CSV, la validation du lot et l’écriture du rapport. Utilise Validator pour les contraintes et une règle de lot pour les doublons. Le service de lecture renvoie soit les données vérifiées, soit une liste d’erreurs identifiées par ligne/champ/code. La commande traduit ce résultat en sortie Console.

Prépare un second fichier :

```csv
reference,title,seats
A-01,HTTP,12
A-02,Symfony,0
A-01,Tests,4
A-04,,5
```

**Validation :** exactement trois problèmes attendus pour cette fixture : ligne 3/champ `seats`, ligne 4/champ `reference` dupliqué, ligne 5/champ `title`. Trier les erreurs par ligne puis champ pour obtenir un résultat stable. La première occurrence d’une référence reste la référence de comparaison ; elle n’est pas signalée comme doublon.

Un en-tête différent ou un nombre de colonnes incorrect est une erreur de structure. Le service ne tente pas de décaler silencieusement les colonnes ou d’inventer des valeurs. Les tests appellent le service réel avec des fichiers temporaires.

## Jalon 2 — Rapport déterministe

Le rapport contient `generatedAt`, `count`, `totalSeats`, `items`. Les items gardent l’ordre du CSV et contiennent `reference`, `title`, `seats`. `seats` est un entier JSON. Utilise `ClockInterface` pour la date et fige l’horloge à `2026-10-11T10:00:00+00:00` dans les tests.

Pour le lot valide : `count = 3`, `totalSeats = 24`, trois items et date exacte. Définis la représentation de date avec fuseau et garde-la stable. Écris le fichier uniquement après validation complète, avec Symfony Filesystem. Vérifie son contenu décodé, pas seulement son existence.

**Validation :** un fichier cible déjà présent avec contenu `ancien rapport` reste identique après un lot invalide. Un succès le remplace par le rapport valide. Le répertoire temporaire du test est nettoyé même en cas d’échec d’assertion.

L’écriture préparée puis remplacée évite un rapport partiel, mais ne constitue pas une exclusion mutuelle entre deux imports. La concurrence est hors de ce laboratoire. Une extension qui exige une exclusion doit employer `symfony/lock`, installé via Composer/Flex, et préciser le stockage de verrous ; pas de drapeau maison.

## Jalon 3 — Contrat Console

Enregistre la commande via `#[AsCommand]`. Le chemin d’entrée est un argument obligatoire ; `--output` attend une valeur. Les deux chemins doivent être distincts : refuse un fichier de sortie qui désigne l’entrée, y compris lorsqu’une normalisation simple révèle leur identité. La commande doit fonctionner avec `--no-interaction` et ne jamais attendre une confirmation.

| Situation | Statut | Sortie / effet |
|---|---|---|
| Lot valide | `Command::SUCCESS` (0) | Résumé sur stdout et rapport écrit |
| Même lot avec `--dry-run` | 0 | Résumé signalant la simulation ; aucun fichier créé/remplacé |
| Option de sortie manquante ou chemins identiques | `Command::INVALID` (2) | Explication, entrée inchangée |
| Fichier d’entrée absent | `Command::FAILURE` (1) | Diagnostic lisible sur stderr ; aucune sortie créée |
| Lot invalide | 1 | Erreurs de lignes sur stderr ; sortie précédente conservée |
| Écriture impossible simulée | 1 | Erreur explicite ; aucun succès annoncé |

Le résumé de succès donne le nombre de lignes valides et de places. Les diagnostics restent sur stderr pour ne pas mélanger données et erreurs. Avec `CommandTester`, active la capture séparée de stderr pour les assertions ; `ApplicationTester` sert à tester le parsing complet et les événements d’application si tu ajoutes cette vérification.

Ne teste pas les permissions avec une hypothèse « chmod suffit » sous n’importe quel utilisateur : double la frontière d’écriture ou prépare un échec déterministe, par exemple une cible dont le parent est un fichier ordinaire.

## Incident à diagnostiquer

La commande retourne 0 sur le lot invalide parce qu’elle a écrit un rapport après chaque ligne correcte et ignoré les erreurs. Construis le test avec une sortie préexistante, exécute le lot invalide et montre les deux régressions : statut trompeur et modification partielle du rapport. Corrige la frontière entre validation et publication, puis garde ce test.

## Indices progressifs

<details>
<summary>Indice 1 — Distinguer donnée et effet</summary>

Lire et préparer le rapport peut être identique en simulation et en exécution. La différence doit se trouver au moment de publier. Un `--dry-run` utile exerce les mêmes validations que le vrai import.

</details>

<details>
<summary>Indice 2 — APIs à observer</summary>

Consulte `SplFileObject::fgetcsv()` ou `fgetcsv()`, `MockClock`, `Filesystem::dumpFile()` et `CommandTester::getErrorOutput()`. Une conversion en entier n’est pas une validation du texte CSV. Un générateur peut lire progressivement, mais le contrat « publier après validation du lot entier » reste à préserver.

</details>

## Livraison personnelle

- [ ] Trois erreurs exactes du lot invalide et préservation de la cible.
- [ ] Rapport valide vérifié avec date figée et types JSON attendus.
- [ ] Statuts, stderr et simulation couverts par des tests Console.
- [ ] Limite de concurrence et stratégie de nettoyage expliquées.

Références : [Console et tests](https://symfony.com/doc/8.0/console.html#testing-commands), [Filesystem](https://symfony.com/doc/8.0/components/filesystem.html), [Clock](https://symfony.com/doc/8.0/components/clock.html), [paramètres de fgetcsv en PHP](https://www.php.net/manual/fr/function.fgetcsv.php).
