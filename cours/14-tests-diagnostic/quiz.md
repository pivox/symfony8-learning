# Quiz — Module 14 : tests et diagnostic

[Parcours](../README.md) · [Cours](README.md) · [Exercices](exercices.md)

API Symfony du socle 8.0 ; syntaxe PHPUnit 13.3 du dépôt. Réponses simples ou multiples explicitement indiquées.

**Q1 (une réponse).** Pour un objet valeur pur, la base adaptée est :
A. TestCase · B. WebTestCase obligatoirement · C. Un client navigateur JavaScript · D. Aucun test possible

**Q2 (une réponse).** Selon les conventions du parcours, un service de l'application se teste avec :
A. Uniquement une capture d'écran · B. KernelTestCase · C. Une requête SQL obligatoire · D. Un test qui vérifie seulement son existence

**Q3 (plusieurs réponses).** Un test HTTP utile peut vérifier :
A. Le statut · B. Le Content-Type · C. Le contenu métier · D. Seulement l'absence d'exception

**Q4 (une réponse).** `createClient()` dans WebTestCase :
A. Exige toujours `bootKernel()` juste avant · B. Démarre déjà le kernel · C. Nécessite `symfony serve` · D. Lance Chromium

**Q5 (une réponse).** Le conteneur de test :
A. Recrée tous les services supprimés par compilation · B. Permet l'accès à des services privés conservés · C. Désactive l'autowiring · D. Ignore l'environnement test

**Q6 (plusieurs réponses).** Pour remplacer une dépendance par un double :
A. Utiliser l'identifiant réellement injecté · B. Remplacer avant d'initialiser le consommateur · C. Vérifier les alias si nécessaire · D. Modifier directement vendor

**Q7 (une réponse).** Un stub se distingue d'un mock notamment parce que :
A. Il fournit des réponses contrôlées sans nécessairement vérifier les appels · B. Il nécessite Internet · C. Il est toujours une classe Doctrine · D. Il garantit toute l'intégration

**Q8 (une réponse).** Une suite sensible à l'ordre d'exécution révèle souvent :
A. Un problème à résoudre par ordre alphabétique obligatoire · B. Une fuite d'état ou fixture partagée · C. Une propriété normale de tous les tests · D. Un besoin de supprimer les assertions

**Q9 (plusieurs réponses).** KernelBrowser :
A. Exécute le kernel sans serveur réseau · B. Redémarre normalement le kernel entre requêtes · C. Exécute tout JavaScript de page · D. Peut envoyer un HTTP_HOST de test

**Q10 (une réponse).** `disableReboot()` garantit-il l'absence de tout reset de service ?
A. Oui, toujours · B. Non, des services peuvent encore être réinitialisés · C. Oui, sauf les DTO · D. Il ne concerne que les cookies

**Q11 (une réponse).** Pour voir l'exception PHP derrière une réponse 500 en test :
A. `catchExceptions(false)` · B. Supprimer la route · C. Ignorer le corps HTTP · D. Transformer tous les statuts en 200

**Q12 (plusieurs réponses).** Pour utiliser le profiler dans ce dépôt initial :
A. Installer d'abord profiler-pack pour cet atelier facultatif · B. Vérifier la configuration en environnement test · C. Activer avant la requête et vérifier qu'un profil existe · D. Supposer un collecteur Doctrine

**Q13 (une réponse).** Un provider PHPUnit moderne doit ici :
A. Utiliser obligatoirement une annotation docblock · B. Être public static et référencé par attribut · C. Démarrer systématiquement un kernel · D. Produire une capture d'écran

**Q14 (plusieurs réponses).** Un bon test de régression :
A. Échoue avant la correction pour la cause visée · B. Passe après correction · C. Peut être minimal et ciblé · D. Remplace toute vérification des frontières affectées

**Q15 (une réponse).** Quelle affirmation est correcte ?
A. 100 % de lignes couvertes prouve l'absence de bug · B. assertInstanceOf suffit à tester une recherche · C. Les assertions sur résultats observables déterminent ce que le test prouve · D. Un test passant une fois garantit son isolation

## Corrections raisonnées

1. **A.** Un objet pur se construit directement. B et C ajoutent des frontières inutiles ; D est faux, ces objets sont précisément simples à tester sans framework.
2. **B.** KernelTestCase vérifie l'appel via la configuration de service réelle. A ne démontre pas la règle, C suppose une base absente et D ne vérifie aucun résultat utile.
3. **A, B, C.** Ces dimensions définissent la réponse reçue par le client. D seul n'établit ni le bon statut ni la bonne représentation ; le kernel peut même convertir une exception en réponse.
4. **B.** Le client prend en charge le démarrage. A peut provoquer une erreur de cycle de kernel ; C est inutile pour ce client interne ; D confond BrowserKit avec un navigateur réel.
5. **B.** Le conteneur spécial expose des services privés retenus. A est faux pour ceux supprimés/inlinés ; C et D contredisent son rôle de conteneur configuré pour les tests.
6. **A, B, C.** Le moment et la cible du remplacement déterminent son effet. D altère les dépendances pour tous les tests et ne constitue pas un double isolé.
7. **A.** Le stub contrôle les réponses ; les attentes d'interaction caractérisent le mock. B et C n'ont aucun rapport ; D promet plus que ce que prouve un double.
8. **B.** État statique, fichiers ou services partagés sont des suspects courants. A masque le problème, C normalise un défaut et D supprime la preuve au lieu de réparer l'isolation.
9. **A, B, D.** Le client exécute PHP avec un contexte serveur configurable et un cycle de kernel. C est faux : aucun moteur JavaScript n'est intégré à BrowserKit.
10. **B.** Des mécanismes comme kernel.reset peuvent toujours s'appliquer. A et C donnent une garantie fausse ; D réduit à tort cette méthode à la gestion des cookies.
11. **A.** L'exception remonte directement au test et expose sa pile. B change le sujet, C cache l'information et D fabrique un succès sans résoudre la cause.
12. **A, B, C.** Le paquet n'est pas initialement présent et la collecte doit être configurée/activée. D est faux : aucun Doctrine n'est installé et les collecteurs dépendent des composants présents.
13. **B.** C'est la syntaxe utilisée avec PHPUnit 13.3 ici. A reprend un ancien modèle ; C mélange préparation de données et intégration ; D ne décrit pas un provider.
14. **A, B, C.** Le test localise et protège le défaut. D est faux : un contrat HTTP ou Console peut encore nécessiter ses propres assertions de statut et de représentation.
15. **C.** Un test prouve les propriétés qu'il observe réellement. A confond exécution et vérification, B ne vérifie aucune recherche et D ne démontre pas l'indépendance à l'ordre.
