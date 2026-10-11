# Approfondissement — Résolution ciblée et réponse différée

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Prérequis : cycle HttpKernel, `RequestStack`, interface et attributs ; prévoir 40 minutes.
Les mécanismes étudiés sont disponibles dans le socle **Symfony 8.0** ; le code installé est **8.1.8**.
Ils complètent les traces du cours en distinguant choix d'un argument et moment d'exécution d'un flux.

## 1. Refuser un argument n'est pas passer la main

Le résolveur reçoit une `Request` et des métadonnées : nom, type, attributs, nullabilité et valeur par défaut.
Sa responsabilité est de fournir les valeurs de cet argument, avant l'appel du contrôleur.
Trois résultats expriment des décisions différentes :

| Résultat du résolveur | Sens | Suite de l'exécution |
|---|---|---|
| `[]` | Je ne fournis aucune valeur | Le résolveur suivant peut essayer |
| `[null]` | La valeur choisie est `null` | La recherche s'arrête pour cet argument |
| `[new ValueObject(...)]` | Valeur résolue | Passage à l'argument suivant |
| Exception HTTP | Cette entrée est refusée | Cycle d'exception, contrôleur non appelé |
| Deux valeurs, argument non variadique | Contrat du résolveur invalide | Exception du framework |

Ne pas retourner `[]` quand un identifiant explicitement fourni est invalide et doit provoquer 404.
Un résolveur suivant pourrait produire une valeur par défaut, transformant silencieusement le refus en succès.
En revanche, un argument d'un autre type doit laisser la main : le résolveur n'en est pas responsable.
Une exception HTTP sort normalement de la chaîne ; les exceptions internes de type « near miss » constituent un mécanisme particulier du framework.

Le ciblage limite ce comportement à un paramètre volontairement marqué.
Il évite qu'un nouveau résolveur modifie toutes les actions possédant un type commun.

```php
use Symfony\Component\HttpKernel\Attribute\AsTargetedValueResolver;
use Symfony\Component\HttpKernel\Attribute\ValueResolver;

// Sur une classe implémentant ValueResolverInterface :
#[AsTargetedValueResolver('lesson.currency')]
// Sur un argument d'action :
// #[ValueResolver('lesson.currency')] Currency $currency
```

L'extrait montre seulement les deux points de déclaration ; il ne constitue pas une classe complète.
Le tag du résolveur ciblé est `controller.targeted_value_resolver`.
Un attribut `ValueResolver` ne crée pas le service : sa définition doit être découverte par l'autoconfiguration.
Dans `ArgumentResolver`, le choix ciblé conserve aussi les recours vers les attributs de requête et les valeurs par défaut.
Il ne faut donc pas interpréter `[]` comme « cet argument est définitivement interdit ».
Deux résolveurs explicitement épinglés sur le même argument sont une erreur de configuration.

Pour un montant en devise, garder séparés extraction de la chaîne et invariant du value object.
Le résolveur convertit une entrée HTTP en argument ; un service métier peut recevoir ce même objet depuis une commande.
Ne pas lui faire exécuter le paiement ou un autre effet métier pendant la résolution.

## 2. Une réponse streamée exécute son contenu à l'envoi

`handle()` produit une `Response` ; `sendContent()` en envoie les octets.
Avec `StreamedResponse`, construire la réponse n'exécute pas le callback qui produit les données.
Le cycle `kernel.response` et `kernel.finish_request` peut donc être terminé avant le début du flux.

```php
use Symfony\Component\HttpFoundation\StreamedResponse;

$response = new StreamedResponse(static function (): void {
    echo "ligne A\n";
    echo "ligne B\n";
});

$response->getContent(); // false : le corps n'est pas une chaîne déjà construite.
```

`sendContent()` n'exécute ce callback qu'une fois pour cette instance de réponse.
Cela ne garantit pas une seule exécution de la route si un client refait sa requête.
Un flux réduit la matérialisation côté application ; PHP, le serveur ou le proxy peuvent encore tamponner les octets.
Un `flush()` ne prouve donc pas que le navigateur affiche déjà la première ligne.

### La pile de requêtes pendant le flux

Après un `handle()` direct terminé, sa requête est dépilée.
Pour une `StreamedResponse`, HttpKernel enveloppe toutefois le callback : il réempile la requête pendant l'envoi, puis la retire dans un `finally`.
Ainsi, un service lisant `RequestStack` pendant ce callback peut retrouver la requête associée au flux.
Ce réempilement ne rejoue pas `kernel.request`, le routage ni la résolution des arguments.
La garantie suppose une réponse passée par `HttpKernel::handle()` ; un objet `StreamedResponse` créé et envoyé seul n'obtient pas ce contexte automatiquement.

| Moment observé, pile initialement vide | Requête courante | Travail du callback |
|---|---|---|
| Contrôleur | Requête principale | Pas encore exécuté |
| Après `handle()` direct | `null` | Pas encore exécuté |
| Dans le callback enveloppé | Requête du flux | En cours |
| Après `sendContent()` | `null` | Terminé ou interrompu |

### Une erreur tardive ne se traite pas comme un refus initial

Une exception au milieu du callback survient après le retour normal de `handle()`.
Elle ne repasse pas dans son `catch` ordinaire ; un test de `kernel.exception` pendant le contrôleur ne couvre pas cette situation.
Lorsque des octets ou en-têtes sont déjà envoyés, reconstruire une erreur JSON complète n'est plus une promesse réaliste.
Valider les préconditions avant de créer le flux ; définir séparément ce que signifie un export interrompu.
Dans un test direct, capturer l'exception de `sendContent()` et les octets déjà produits.
Vérifier aussi le nettoyage de la pile, sans ajouter soi-même un `pop()` correctif.

## Observer dans l'application

```bash
bin/console debug:container --tag=controller.targeted_value_resolver
bin/console debug:container --tag=controller.argument_value_resolver
bin/console debug:event-dispatcher kernel.controller_arguments
php bin/phpunit tests/Architecture
```

Sources locales : `vendor/symfony/http-kernel/Controller/ArgumentResolver.php`, `HttpKernel.php` et `vendor/symfony/http-foundation/StreamedResponse.php`.
Pour séparer les phases, appeler le Kernel puis `sendContent()` sous tampon de sortie dans un `try/finally` de test.
BrowserKit peut capturer le flux lors de son propre cycle ; consigner le montage retenu avant d'interpréter une trace.
Le profiler n'est pas nécessaire : collecteur de traces et assertions sur les octets suffisent.

## Documentation officielle

- [Cycle du composant HttpKernel — Symfony 8.0](https://symfony.com/doc/8.0/components/http_kernel.html).
- [HttpFoundation et réponses streamées — Symfony 8.0](https://symfony.com/doc/8.0/components/http_foundation.html#streaming-a-response).
- [Source du résolveur d'arguments — branche officielle 8.0](https://github.com/symfony/http-kernel/blob/8.0/Controller/ArgumentResolver.php).
