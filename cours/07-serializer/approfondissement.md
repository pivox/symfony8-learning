# Approfondissement — états absents et mises à jour

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Les types, groupes et erreurs d'entrée du cours restent le point de départ.
Cette fiche traite deux autres contrats : la forme exacte d'une réponse vide et la modification d'un objet déjà construit.
Socle **Symfony 8.0**, vérifié dans le code installé **8.1.8** ; aucune API propre à 8.1 n'est nécessaire ici.
Prérequis : normalisation, propriétés typées PHP, contexte et tests de service.

## 1. Absent, null et vide ne disent pas la même chose

Une propriété nullable initialisée à `null` existe dans l'objet.
Une propriété typée sans valeur n'est pas initialisée : tenter une lecture PHP ordinaire peut échouer.
Un tableau vide est une valeur définie ; il peut représenter une collection sans éléments.
Le client JSON doit pouvoir distinguer ces situations selon le contrat annoncé.
Une clé omise peut signifier « information non exposée », tandis que `null` signifie « absence explicitement connue ».
Le Serializer n'invente pas cette sémantique métier : tu choisis le contexte et la documentes.

```php
use Symfony\Component\Serializer\Normalizer\AbstractObjectNormalizer;

$context = [AbstractObjectNormalizer::SKIP_NULL_VALUES => true];
// À passer à normalize() ou serialize() sur le DTO de réponse choisi.
```

`SKIP_NULL_VALUES` concerne la normalisation ; ce n'est pas une règle qui ignorerait les `null` reçus lors d'une mise à jour.
Par défaut, `ObjectNormalizer` omet les propriétés typées non initialisées.
Désactiver `SKIP_UNINITIALIZED_VALUES` permet de rendre leur lecture défaillante observable.
Ce réglage peut révéler un DTO incomplet, mais ne remplace pas une construction correcte de ce DTO.
Un getter exécuté pendant la normalisation reste du code PHP : il peut lui aussi lever une exception.

| État du champ | Sortie habituelle | Décision à expliciter |
|---|---|---|
| `note = null` | `"note": null` | garder l'information ou omettre avec `SKIP_NULL_VALUES` |
| `tags = []` | `"tags": []` | préserver une collection vide |
| propriété typée non initialisée | omise par défaut | initialiser ou rendre l'erreur visible en diagnostic |
| objet sans attribut normalisé | peut devenir `[]` | préserver sa nature objet avec `PRESERVE_EMPTY_OBJECTS` |

`PRESERVE_EMPTY_OBJECTS` garde un objet vide sous la forme JSON `{}`.
Il vise les objets normalisés ; il ne demande pas de transformer toutes les collections vides en objets.
Le contexte `Serializer::EMPTY_ARRAY_AS_OBJECT` a une portée différente : il agit sur les tableaux vides normalisés.
Choisir cette dernière option globalement peut donc modifier le contrat d'une collection attendue en `[]`.
Un groupe qui retire tous les attributs d'un objet est un bon cas pour observer cette différence.

Un test comparant seulement deux tableaux PHP après `json_decode(..., true)` perd la distinction `{}`/`[]`.
Pour protéger la forme, décoder aussi en objets ou comparer un JSON attendu qui préserve ces types.
Vérifier `array_key_exists()` lorsqu'une clé à `null` doit être distinguée d'une clé absente.
`isset()` retourne faux dans ces deux cas et ne suffit donc pas à prouver le contrat.

## 2. Hydrater un objet existant n'est pas fusionner arbitrairement un graphe

Par défaut, une désérialisation construit un objet neuf.
`AbstractNormalizer::OBJECT_TO_POPULATE` fournit une instance existante pour la racine.
Les attributs absents de l'entrée ne deviennent pas automatiquement `null` : ils ne sont pas écrits.
Les attributs présents passent par les accès d'écriture disponibles, avec leurs types et leurs règles.
Cette opération convient à un objet mutable ; elle ne constitue pas une stratégie de modification d'un DTO `readonly` déjà initialisé.
Pour un objet immuable, préférer un DTO d'entrée distinct puis construire une nouvelle valeur dans le service.

```php
use Symfony\Component\Serializer\Normalizer\AbstractNormalizer;

$context = [AbstractNormalizer::OBJECT_TO_POPULATE => $existingDraft];
// $existingDraft est ici un objet mutable fourni explicitement par l'appelant.
// Comparer ensuite les valeurs ET l'identité de l'objet retourné.
```

La racine conservée n'implique pas que ses sous-objets le soient.
Sans option supplémentaire, un sous-objet reçu peut être reconstruit et remplacer l'ancien.
`AbstractObjectNormalizer::DEEP_OBJECT_TO_POPULATE` permet de réutiliser les sous-objets existants.
Ce comportement concerne les objets individuels, pas une fusion de collections par identifiant.
Un tableau d'objets fourni à nouveau n'est pas rapproché automatiquement de ses anciennes lignes.
Un champ `id` n'est pas un ordre de charger une entité ; aucune persistence n'intervient dans ces ateliers.

| Besoin | Outil adapté | Limite à tester |
|---|---|---|
| conserver l'instance racine mutable | `OBJECT_TO_POPULATE` | identité racine, champs omis conservés |
| conserver l'adresse déjà attachée | ajouter `DEEP_OBJECT_TO_POPULATE` | identité et valeurs du sous-objet |
| modifier une collection par identifiant | service métier explicite | ajouts, retraits et doublons définis par contrat |
| modifier une valeur immuable | nouveau DTO et nouvelle instance | ancienne instance inchangée |

La dénormalisation écrit progressivement ; une exception tardive n'annule pas les écritures déjà effectuées.
Il ne faut donc pas traiter `OBJECT_TO_POPULATE` comme une transaction.
Pour une opération devant être entièrement acceptée ou refusée, travailler sur une entrée distincte et valider avant l'application métier.
Un test peut volontairement fournir un champ correct puis un champ de type impossible pour observer cette limite.
Ne pas rendre l'atomicité dépendante de l'ordre des clés d'un document JSON.

## 3. Observer dans l'application

```bash
symfony console debug:config framework serializer
symfony console debug:container serializer.normalizer.object
symfony console debug:serializer 'App\Serializer\ProfileOutput'
php bin/phpunit tests/Serializer
```

La commande de métadonnées suppose d'avoir créé la classe indiquée ; adapter son nom à l'atelier.
Dans `KernelTestCase`, appeler le Serializer du conteneur avec une fixture neuve pour chaque contexte.
Capturer l'objet enfant avant l'appel permet de comparer `===` après une mise à jour profonde.
Pour une réponse HTTP, contrôler aussi les clés absentes dans le document réellement renvoyé.
Le passage par `MapRequestPayload` reste la voie HTTP ; ces expériences de mise à jour sont d'abord des appels de service.
En cas d'exception inattendue, lire `var/log/dev.log` et la trace ; le profiler n'est pas installé.

## Sources et lecture ciblée

- [Serializer 8.0 : objets vides, valeurs nulles et objets existants](https://symfony.com/doc/8.0/serializer.html).
- [Normalizers 8.0](https://symfony.com/doc/8.0/serializer/normalizers.html) pour distinguer les normalizers et leurs contextes.
- Source installée : `vendor/symfony/serializer/Normalizer/AbstractNormalizer.php`, extraction de `OBJECT_TO_POPULATE`.
- Source installée : `vendor/symfony/serializer/Normalizer/AbstractObjectNormalizer.php`, lecture des valeurs et contexte enfant.

Avant de poursuivre, expliquer pourquoi « même JSON décodé en tableau » ne prouve pas « même forme JSON ».
Puis relier la conservation d'identité à un besoin concret, plutôt que d'activer toutes les options de mise à jour.
