# Quiz — Module 16 : Cache applicatif

[Cours](README.md) · [Exercices](exercices.md)

Questions d'entraînement originales, socle Symfony 8.0.

**Q1 (une réponse).** Quel cache concerne ce module ?
A. Réponses d'un proxy B. Valeurs calculées par les services C. Uniquement les cookies D. Le DNS

**Q2 (une réponse).** Avec les Cache Contracts, le callback de `get()` fournit :
A. Le contrôleur B. La clé seulement C. La valeur à mémoriser D. Une réponse HTTP obligatoire

**Q3 (une réponse).** Un item PSR-6 contenant `null` peut-il être un hit ?
A. Oui B. Non C. Seulement en HTTP D. Seulement avec Doctrine

**Q4 (une réponse).** Quel appel distingue présence et absence en PSR-6 ?
A. `get() === null` B. `empty()` C. `isset($pool)` D. `isHit()`

**Q5 (une réponse).** Une fiche traduite exige dans sa clé :
A. Uniquement le produit B. Le produit et la langue C. La langue seulement D. La date du dernier déploiement seulement

**Q6 (une réponse).** Pour tester le TTL par défaut d'ArrayAdapter sans attente ni expiration explicite de l'item :
A. `sleep(60)` B. Changer seulement la date du service C. Lui fournir une horloge contrôlée D. Vider le cache avant chaque lecture

**Q7 (une réponse).** Les tags nécessitent :
A. Un pool compatible B. Doctrine C. Twig D. Un cookie

**Q8 (une réponse).** `invalidateTags(['product.42'])` vise :
A. Toutes les données du serveur B. Seulement une entrée nommée `product.42` C. Les classes PHP D. Les entrées associées à ce tag

**Q9 (une réponse).** `beta: 0` signifie :
A. Aucune mémorisation B. Pas d'expiration anticipée probabiliste C. Expiration immédiate D. Durée de vie infinie

**Q10 (une réponse).** `beta: INF` demande :
A. Un timeout infini B. Une erreur C. Un recalcul forcé D. Une suppression de tous les pools

**Q11 (une réponse).** Un callback de cache doit-il débiter une carte ?
A. Non : le recalcul peut se répéter B. Oui, une fois seulement C. Oui, si la clé est courte D. Oui, en mémoire

**Q12 (une réponse).** Deux pools peuvent-ils contenir la même clé logique ?
A. Non B. Seulement en production C. Seulement sans TTL D. Oui, dans des espaces isolés

**Q13 (une réponse).** Symfony Contracts et PSR-16 :
A. Ont nécessairement la même API B. Ont des interfaces distinctes malgré le même nom court C. Sont uniquement du cache HTTP D. Exigent Redis

**Q14 (une réponse).** Un test ArrayAdapter prouve-t-il une exclusion mutuelle distribuée ?
A. Oui B. Oui avec deux clés C. Non D. Oui avec `beta: INF`

**Q15 (plusieurs réponses).** Un bon test d'invalidation vérifie :
A. La fraîcheur du groupe modifié B. Le recalcul forcé de tous les groupes C. Une attente réelle D. L'absence de recalcul inutile des autres groupes

## Réponses

1. **B** — Le cache HTTP appartient au module 05. Cookies et DNS ne sont pas les valeurs étudiées ici.
2. **C** — Le callback retourne le résultat et peut configurer l'item reçu. Il n'a aucune obligation de produire une réponse HTTP.
3. **A** — `null` est une valeur valide ; ni HTTP ni Doctrine ne détermine le hit.
4. **D** — Tester la valeur ou le pool ne distingue pas une entrée présente contenant `null` d'une absence.
5. **B** — Les deux paramètres influencent le résultat. Les autres clés mélangent produits ou langues.
6. **C** — Le pool utilise son horloge pour son TTL par défaut. Une expiration explicite via `expiresAfter()` lit encore le temps réel dans la version installée : utiliser `expiresAt()` calculé avec l'horloge du test si nécessaire. A ralentit le test ; B ne contrôle pas le pool ; D ne teste pas l'échéance.
7. **A** — Les tags sont une capacité du pool, indépendante de Doctrine, Twig et des cookies.
8. **D** — Un tag n'est pas une clé unique et ne vide pas nécessairement tout le cache.
9. **B** — Le TTL normal continue à s'appliquer ; les autres réponses confondent les paramètres.
10. **C** — L'effet porte sur l'entrée demandée, pas sur un timeout ni sur l'ensemble des pools.
11. **A** — Éviction, expiration et concurrence peuvent répéter le callback ; ni clé ni adaptateur mémoire ne garantit un effet unique.
12. **D** — L'isolation est précisément l'intérêt des pools ; le TTL et l'environnement ne l'interdisent pas.
13. **B** — Vérifie le namespace. Les deux API n'imposent ni protocole HTTP ni backend Redis.
14. **C** — Un test dans un processus ne reproduit pas un stockage partagé avec plusieurs workers.
15. **A, D** — Il faut vérifier correction et ciblage. Recalculer tout masque une invalidation trop large ; dormir n'en apporte pas la preuve.
