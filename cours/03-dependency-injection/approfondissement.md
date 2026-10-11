# Approfondissement — Durée de vie et résolution différée

[Cours](README.md) · [Exercices](exercices.md) · [Quiz](quiz.md)

Prérequis : constructeur, alias, container compilé et lazy ; prévoir 35 minutes.
Les exemples utilisent les API **Symfony 8.0**, vérifiées dans les sources **8.1.8** installées.
La remise à zéro des services non partagés possède une extension 8.1 explicitement signalée plus bas.

## 1. `shared: false` ne veut pas dire « neuf à chaque méthode »

Par défaut, le container partage une instance d'un service dans sa propre durée de vie.
Une dépendance injectée au constructeur reste celle détenue par le consommateur.
Ce principe vaut même si la définition de la dépendance est non partagée.
Le container ne revient pas inspecter chaque appel de méthode pour remplacer ses propriétés.

```php
use Symfony\Component\DependencyInjection\Attribute\Autoconfigure;

#[Autoconfigure(shared: false)]
final class DraftBuffer
{
    private array $lines = [];

    public function append(string $line): void
    {
        $this->lines[] = $line;
    }
}
```

Si `Exporter` partagé reçoit `DraftBuffer` une fois, ses deux appels utilisent ce même buffer.
Deux consommateurs construits séparément peuvent, eux, recevoir deux buffers différents.
Le point de création est une résolution de dépendance, pas le début arbitraire d'une opération métier.
Écrire un test sur l'identité des instances permet de distinguer ces deux situations.
Puis tester le contenu produit : l'identité seule n'explique pas toutes les fuites d'état.

| Besoin | Choix possible | Limite |
|---|---|---|
| Service sans état propre à une opération | Partagé, comportement pur | Aucun état de requête conservé |
| Un buffer par consommateur construit | Définition non partagée | Le consommateur peut vivre longtemps |
| Un buffer par export | Création au début de l'opération | Il faut un point de création explicite |
| Client lourd rarement utilisé | Lazy ou accès différé | L'état peut encore être partagé après création |

Un objet de travail très simple peut être construit avec `new` dans un service ou une fabrique typée.
Il n'est pas obligatoire de faire de chaque DTO ou collection mutable un service autowiré.
Le container compose les dépendances ; il ne remplace pas le modèle de durée de vie du domaine.

## 2. Une closure diffère la résolution, pas nécessairement la création

`AutowireServiceClosure` injecte une closure qui demande un service lorsqu'on l'appelle.
Elle ne transforme pas automatiquement une définition partagée en fabrique d'instances neuves.

```php
use Symfony\Component\DependencyInjection\Attribute\AutowireServiceClosure;

public function __construct(
    #[AutowireServiceClosure(DraftBuffer::class)]
    private \Closure $bufferFactory,
) {}
```

Cet extrait est un constructeur de consommateur ; son opération décide quand invoquer la closure.
Pour une définition partagée, deux appels à la closure retrouvent normalement la même instance.
Pour une définition non partagée, ces appels résolvent deux instances distinctes.
Appeler la closure dès le constructeur annule l'intérêt de différer cette résolution.
Conserver son premier résultat dans une propriété transforme aussi les demandes suivantes en réutilisation locale.

Le locator est adapté lorsqu'une clé connue au moment de l'appel sélectionne une dépendance.
Sa surface doit rester limitée aux services déclarés ; un locator restreint n'est pas le container global.
Un service absent se distingue d'un service dont la construction échoue : `has()` ne valide pas tout son graphe d'exécution.
Tester `has('csv')` ne prouve donc pas que l'export CSV fonctionne.

L'itérateur différé répond à une autre question : parcourir plusieurs services selon leurs priorités.
Le fait qu'il soit réitérable ne signifie pas que tous les services visités seront recréés au second passage.
Cette propriété dépend encore de leurs définitions partagées ou non partagées.

| Mécanisme | Décision reportée | Question à poser au test |
|---|---|---|
| Lazy | Initialisation du service | L'état a-t-il déjà été touché ? |
| Closure de service | Résolution à l'appel | Même définition et quel partage ? |
| Locator | Choix du service par clé | Quelle clé a réellement été demandée ? |
| Fabrique métier | Construction d'un objet de travail | Une opération reçoit-elle son propre objet ? |

## 3. Remise à zéro et objet neuf sont deux contrats différents

Un service partagé peut volontairement collecter un état temporaire : traces, compteurs ou contexte.
`Symfony\Contracts\Service\ResetInterface` exprime sa capacité à nettoyer cet état via `reset()`.
L'autoconfiguration associe les services concernés au mécanisme `kernel.reset`.
La remise à zéro est déclenchée par le cycle du framework qui réutilise ces services ; appeler une méthode métier ne la déclenche pas.

Un reset conserve l'instance mais efface l'état désigné ; reconstruire un container crée un autre graphe d'objets.
Avec un client HTTP de test, un redémarrage peut masquer une fuite présente dans un processus réutilisé.
Pour diagnostiquer, relever séparément l'identité, les données avant reset et les données après reset.
`reset()` doit être idempotent : deux appels successifs laissent le service dans le même état vide.
Il ne doit pas effacer les dépendances injectées ni modifier une configuration immuable.

**Complément 8.1** : le changelog DI ajoute la remise à zéro des services non partagés marqués `kernel.reset`.
Ne pas faire de cette extension une hypothèse du socle de certification 8.0.
Pour le socle, étudier d'abord un collecteur partagé avec un reset observable.
Une fabrique d'objets de travail reste souvent plus simple qu'un service mutable à nettoyer partout.

## Observer dans l'application

```bash
bin/console debug:container --tag=kernel.reset
bin/console debug:container 'App\Di\DraftBuffer'
bin/console lint:container --env=test
php bin/phpunit tests/Di
```

La commande ciblant `DraftBuffer` s'utilise une fois l'exemple créé dans l'exercice.
Lire les factories générées dans `var/cache/test/` sans les modifier : repérer stockage d'instance et création directe.
Sources : `Attribute/AutowireServiceClosure.php`, `Attribute/Autoconfigure.php` et `CHANGELOG.md` sous `vendor/symfony/dependency-injection/`.

## Documentation officielle

- [Services non partagés — Symfony 8.0](https://symfony.com/doc/8.0/service_container/shared.html).
- [Subscribers et locators — Symfony 8.0](https://symfony.com/doc/8.0/service_container/service_subscribers_locators.html).
- [Services lazy — Symfony 8.0](https://symfony.com/doc/8.0/service_container/lazy_services.html).
- [Changelog DI officiel 8.1](https://github.com/symfony/dependency-injection/blob/8.1/CHANGELOG.md).
