# Atelier 02 — Un devis, du formulaire au récapitulatif

[Ateliers](README.md) · [Twig et Forms](../09-twig-forms/README.md) · [Validator](../12-validator/README.md) · [Tests](../14-tests-diagnostic/README.md)

**Niveau :** intermédiaire. **Durée :** 3–4 h. **Prérequis :** modules 03, 04, 09, 12 et 14. Code dans `src/Ateliers/Devis/`, tests dans `tests/Ateliers/Devis/`, templates dans `templates/ateliers/devis/`.

## Mission et contrat

Construis un formulaire public de devis pour un atelier de formation. Le devis est conservé uniquement dans la session du visiteur pour l’afficher après redirection. Il n’y a ni base, ni compte, ni email envoyé. Les règles métier ne doivent pas dépendre du formulaire pour être testées.

| Champ | Règle |
|---|---|
| `name` | Nom obligatoire, 2 à 80 caractères après suppression des espaces de bord |
| `email` | Obligatoire, adresse valide |
| `participants` | Entier de 1 à 6 |
| `format` | Choix `remote` ou `onsite` |
| `comment` | Facultatif, au plus 300 caractères |

Le tarif est de **4500 centimes par participant**, avec **3000 centimes supplémentaires par devis** pour `onsite`. Pas de réduction ni de calcul en nombres flottants. Deux participants en présence coûtent donc 12000 centimes ; trois à distance coûtent 13500 centimes.

## Jalon 1 — Modèle et calcul

Crée un modèle adapté au formulaire, un `FormType` et un service de calcul. Un objet de formulaire peut être mutable pour le mapping ; le résultat du calcul est un objet valeur `readonly`. Place les contraintes sur les propriétés ; le service protège aussi ses invariants lorsqu’il est appelé hors HTTP.

**Validation :** tests du service pour `(1, remote)`, `(2, onsite)` et `(6, remote)` ; quantités 0 et 7 et format inconnu refusés. Le résultat expose montant en centimes et quantité. Un calcul n’écrit rien dans la session.

L’adresse est une donnée de formulaire, pas une autorisation de contacter la personne. Les adresses des tests utilisent `example.test`.

## Jalon 2 — Soumettre avec Form et Twig

La route `GET|POST /training/devis` est gérée par un contrôleur fin. Le formulaire est lié à son modèle, traite la requête avec Form et utilise la protection CSRF. Ajoute un identifiant de jeton propre à ce formulaire et un nom racine stable.

Le premier GET rend le formulaire avec labels et bouton `Calculer`. Le POST invalide rend le même formulaire avec statut **422**, valeurs conservées et erreurs lisibles. Un champ supplémentaire `totalCents=1` est refusé ; le prix est toujours recalculé côté serveur.

Pour rendre l’essai CSRF reproductible, choisis pour cet atelier un identifiant de token avec stockage de session, distinct des identifiants éventuellement configurés comme stateless. Observe d’abord `debug:config framework csrf_protection`. Le test du token d’un autre visiteur dépend de ce choix explicite.

**Validation :** le crawler soumet le vrai formulaire ; nom vide, adresse incorrecte, quantité hors limites, champ inconnu et token manquant produisent chacun un refus et aucun nouveau devis. Si une session possède déjà un devis, une tentative invalide ne remplace pas ce devis.

## Jalon 3 — Post/Redirect/Get et isolation

Après soumission valide, stocke seulement les données nécessaires au récapitulatif sous une clé dédiée et retourne **303** vers la route nommée de `GET /training/devis/recapitulatif`. Un accès direct sans devis en session retourne 404. Le récapitulatif affiche nom, format, quantité et montant, avec une réponse `private, no-store`.

Affiche un flash de confirmation sur le premier GET du récapitulatif. Une seconde lecture conserve le devis mais ne réaffiche pas ce flash. Recharger cette page ne soumet pas à nouveau le formulaire et ne recalcule pas un nouveau devis.

| Scénario | Résultat observable |
|---|---|
| Deux participants `onsite` | POST 303, puis récapitulatif de 12000 centimes |
| Nom `<b>Sam</b>` | Texte affiché littéralement, aucune balise `b` issue de ce nom |
| Deuxième GET du récapitulatif | Même devis, flash absent |
| Nouveau navigateur sans cookies | 404 sur le récapitulatif |
| Token CSRF copié depuis un autre visiteur | 422, aucun nouveau devis |
| Injection de `totalCents` | 422, prix fourni par le client jamais utilisé |

Génère les URL dans Twig avec `path()` ; la redirection utilise le nom de route. Dans le test HTTP, une URL littérale est utile pour vérifier le contrat public. Le crawler ne lance pas JavaScript : le formulaire doit fonctionner en HTML natif.

## Incident à diagnostiquer

Un développeur stocke le dernier devis dans une propriété du service partagé. Deux appels successifs avec deux visiteurs peuvent maintenant se contaminer ; le redémarrage du kernel peut aussi faire disparaître le résultat. Explique pourquoi aucun de ces comportements n’est une session.

Construis une expérience dans le même kernel qui révèle l’état retenu, puis vérifie le vrai parcours de deux visiteurs aux cookies distincts. Garde le service de calcul sans état et confie le récapitulatif au stockage de session. Ne désactive pas tous les resets du kernel pour faire passer le test.

## Indices progressifs

<details>
<summary>Indice 1 — Choisir les responsabilités</summary>

Le `FormType` décrit les champs ; le Validator vérifie les données ; le calculateur connaît les centimes ; le contrôleur décide de la réponse et de la session ; Twig affiche. Une propriété de service n’est pas un stockage partagé entre requêtes garanti.

</details>

<details>
<summary>Indice 2 — Prouver la navigation</summary>

Inspecte `Location` avant `followRedirect()`. Récupère le formulaire depuis le crawler pour conserver ses champs cachés. Pour un second visiteur, utilise un parcours avec une autre cookie jar et un cycle de kernel maîtrisé ; ne réutilise pas accidentellement les cookies du premier.

</details>

## Livraison personnelle

- [ ] Calculs et invariants testés par appel du service.
- [ ] Refus du formulaire et absence de remplacement du devis vérifiés.
- [ ] Parcours 303 → GET, flash consommé une fois et HTML échappé.
- [ ] Deux visiteurs isolés et politique de cache du récapitulatif vérifiée.

Références : [formulaires Symfony 8.0](https://symfony.com/doc/8.0/forms.html), [protection CSRF](https://symfony.com/doc/8.0/security/csrf.html), [sessions](https://symfony.com/doc/8.0/session.html). Observe les options du formulaire, les cookies de BrowserKit et les en-têtes de la réponse, sans supposer une configuration de session universelle.
