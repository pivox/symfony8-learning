# Exercices — Module 12 : inscription selon le contexte

[Parcours](../README.md) · [Cours](README.md) · [Quiz](quiz.md)

Code : `src/Validation/`, tests : `tests/Validation/`, exécution : `php bin/phpunit tests/Validation`.
Scénario : DTO d'inscription public, validation uniquement, aucune création de compte ni persistence. HTTP JSON pour le défi final.
Services avec `KernelTestCase`, HTTP avec `WebTestCase`. Initialiser les DTO via un constructeur promu, `readonly` lorsque compatible.
Écrire les cas attendus avant les contraintes ; les indices ne fournissent pas les solutions.

## 1. Un email obligatoire — guidé

Scénario : `SignupInput` contient un email et un nom d'affichage ; le nom est limité à 60 caractères.
Travail : déclarer les contraintes en attributs et appeler le Validator du conteneur.

- Attendus : email `ada@example.test`, nom `Ada` → zéro violation ; email `incorrect` → violation `email`.
- Acceptation : email vide refusé ; nom de 61 caractères refusé ; vérifier codes et chemins, pas uniquement un message.
- Cas limite : comparer `NotNull` et `NotBlank` sur `''` avec une validation de valeur dédiée.
- Indice : une contrainte de format ne rend pas nécessairement le champ obligatoire.

## 2. Brouillon et publication — guidé

Scénario : le nom peut être vide dans un brouillon mais devient obligatoire au moment de publier l'inscription.
Travail : créer `publish`, garder les règles communes dans `Default` et expliciter la sélection dans l'appelant.

- Attendus : email valide + nom vide passe en brouillon, échoue sur `displayName` avec `['Default', 'publish']`.
- Acceptation : email incorrect échoue dans les deux contextes ; nom trop long aussi.
- Cas limite : valider seulement `publish` et expliquer pourquoi les règles communes peuvent manquer.
- Indice : un groupe personnalisé n'inclut pas implicitement `Default`.

## 3. Ordonner les vérifications — semi-guidé

Scénario : une règle `Strict` vérifie un domaine dans une liste locale ; on veut éviter cet appel si l'email est déjà mal formé.
Travail : instrumenter un service de liste autorisée, puis comparer tableau de groupes et `GroupSequence`.

- Attendus : email mal formé → aucune consultation de liste avec séquence ; email bien formé interdit → violation Strict.
- Acceptation : une adresse autorisée déclenche une consultation et passe ; compter les appels avec un double contrôlé.
- Cas limite : comparer le nombre d'appels avec `['Default', 'Strict']`, qui ne crée pas de barrière entre groupes.
- Indice : l'arrêt se fait entre groupes, pas forcément entre chaque contrainte d'un groupe.

## 4. Valider une adresse imbriquée — semi-guidé

Scénario : publication avec `AddressInput` contenant `city` obligatoire et `postalCode` au format de cinq chiffres.
Travail : ajouter cascade et présence obligatoire pour `address`, puis une collection `contacts` de DTO avec email.

- Attendus : ville vide → chemin `address.city` ; premier contact invalide → `contacts[0].email`.
- Acceptation : `address = null` refusé lors de publication ; objet complet valide accepté ; groupes enfants documentés.
- Cas limite : retirer temporairement `Valid` dans un test expérimental et constater la disparition des erreurs enfant.
- Indice : la cascade ne signifie pas automatiquement « valeur non nulle ».

## 5. Construire une contrainte réutilisable — autonome

Scénario : seuls `example.test` et `school.test` sont admis par la règle `AllowedEmailDomain` pour publication.
Travail : créer contrainte attribut et validateur injectant un service de politique locale ; aucune requête réseau.

- Attendus : les deux domaines autorisés passent ; `other.test` produit une violation avec un code stable choisi pour cette règle.
- Acceptation : `null` et vide sont laissés aux contraintes de présence ; le contrat des types inattendus est explicite et testé.
- Observation : retrouver le service via `debug:container --tag=validator.constraint_validator`.
- Indice : la contrainte contient la configuration, le validateur reçoit les services ; étendre `ConstraintValidator`.

## 6. Croiser des propriétés — autonome

Scénario : l'adresse de confirmation doit égaler l'email principal en publication, sans normalisation implicite.
Travail : poser une règle de classe ou un callback et rattacher l'erreur à `confirmationEmail`.

- Attendus : valeurs identiques → aucune erreur de cohérence ; valeurs différentes → exactement un chemin `confirmationEmail`.
- Acceptation : pas de chemin vide ni `confirmationEmail.confirmationEmail` ; comparer aussi le code/message métier.
- Cas limite : décider explicitement si une absence de confirmation relève de présence ou de cohérence et éviter les doublons inutiles.
- Indice : `atPath()` est relatif au contexte dans lequel la violation est construite.

## 7. Réparer une validation silencieuse — diagnostic

Scénario : un payload invalide passe parce que l'appelant demande `publish` seul ; l'adresse passe parce que la cascade manque.
Travail : recréer les deux défauts, puis examiner métadonnées et appel effectif avant toute correction.

- Attendus : un test rouge par défaut et un diagnostic expliquant quelle règle n'a pas été exécutée.
- Acceptation : `debug:validator` montre contraintes et groupes ; les corrections rétablissent les chemins attendus.
- Cas limite : une contrainte `Strict` ne doit pas s'exécuter après un échec du groupe de base dans le scénario séquencé.
- Indice : l'absence de violation peut signifier « règle non sélectionnée », pas « règle correcte ».

## 8. Défi intégrateur — endpoint de prévalidation

Scénario : `POST /lab/signup/check` reçoit un DTO JSON et valide le contexte publication ; réponse nominale `{ "accepted": true }`.
Travail autonome : route en attributs, `MapRequestPayload`, service de validation réutilisable et tests du contrat complet.

- Attendus : inscription complète autorisée → 200 JSON ; email interdit ou adresse invalide → 422 avec erreurs localisables.
- Acceptation : JSON mal formé → 400 ; type incompatible → réponse d'erreur dont statut/structure sont constatés et figés par test.
- Couvrir : brouillon valide via service, publication refusée, cascade, cohérence, aucune donnée enregistrée.
- Livraison : matrice de cas (donnée, contexte, chemins attendus) et preuve qu'HTTP et service appliquent les mêmes règles de publication.
- Indice : déclarer `format: 'json'` sur la route et choisir explicitement les groupes du mapping.
