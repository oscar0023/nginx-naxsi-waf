# Phase 7 — Évaluation des performances

## 1. Objectif

Cette phase vise à évaluer le temps de réponse de l'application DVWA en comparant deux chemins d'accès :

- **Backend direct** : accès direct à Apache/DVWA sur le port 3001.
- **Accès protégé** : accès via NGINX avec le module NAXSI activé, sur le port 80.

L'objectif est d'estimer le surcoût associé au passage par l'architecture du WAF dans les conditions du laboratoire.

## 2. Environnement de test

| Élément | Configuration |
|---|---|
| Serveur WAF | Ubuntu Server 22.04.5 LTS |
| Serveur web / proxy | NGINX 1.18.0 |
| Module WAF | NAXSI 1.7 |
| Application cible | DVWA |
| Backend | `127.0.0.1:3001` |
| Point d'entrée WAF | `127.0.0.1:80` |
| Outil de mesure | curl |
| Nombre de requêtes | 30 par chemin |

## 3. Méthodologie

Les mesures ont été effectuées depuis le serveur Ubuntu afin de limiter les variations dues au réseau entre machines virtuelles.

Les deux chemins ont été testés avec la même ressource HTTP :

- Backend direct : `http://127.0.0.1:3001/`
- Via le WAF : `http://127.0.0.1/`

La commande `curl` a été utilisée pour relever le code HTTP et le temps total de chaque requête.

Les mesures ont ensuite été agrégées avec `awk` pour calculer le temps moyen, minimum et maximum.

## 4. Résultats

| Indicateur | Backend direct | Via NGINX + NAXSI |
|---|---:|---:|
| Nombre de requêtes | 30 | 30 |
| Temps moyen | 7,1 ms | 10,3 ms |
| Temps minimum | 2,1 ms | 3,0 ms |
| Temps maximum | 67,9 ms | 118,8 ms |
| Réponses HTTP 302 | 30 | 30 |

Toutes les requêtes des deux séries ont reçu le code HTTP 302, correspondant à une redirection de l'application DVWA.

## 5. Analyse des résultats

### 5.1 Différence de temps moyen

La différence calculée à partir des moyennes affichées est :

ΔT = 10,3 ms − 7,1 ms = 3,2 ms

### 5.2 Surcoût relatif estimé

Surcoût relatif = ((10,3 − 7,1) / 7,1) × 100

**Surcoût relatif estimé : environ 45,1 %.**

Ce pourcentage est calculé à partir de valeurs arrondies à quatre décimales de seconde. Il représente une comparaison des temps mesurés dans cet essai et non une mesure isolée du coût de NAXSI.

### 5.3 Variabilité

Les temps maximums sont supérieurs aux temps moyens dans les deux séries. Cela indique que les temps de réponse ont varié au cours des essais.

Les causes possibles incluent les variations de charge du système, l'ordonnancement des processus, les opérations du serveur web et les conditions d'exécution des machines virtuelles. Ces causes n'ont pas été isolées individuellement.

## 6. Limites de l'expérience

- Les mesures concernent uniquement la ressource `/`.
- Les deux séries ont renvoyé HTTP 302 ; elles mesurent donc une réponse de redirection et non le chargement complet d'une page DVWA authentifiée.
- Le test est séquentiel et ne représente pas une charge concurrente.
- Le protocole utilisé ne sépare pas le temps consacré à NGINX, à NAXSI et au backend.
- Les résultats proviennent d'une seule série de 30 requêtes par chemin.
- Le pourcentage de surcoût est sensible aux variations et à l'arrondi des moyennes.

## 7. Améliorations possibles

Pour renforcer la validité des résultats, les travaux futurs pourront inclure :

1. Effectuer plusieurs séries après quelques requêtes de chauffe.
2. Comparer les médianes et les percentiles, notamment P95.
3. Mesurer une ressource identique produisant une réponse HTTP 200 dans les deux configurations.
4. Réaliser des essais de charge contrôlés et surveiller l'utilisation CPU et mémoire.
5. Vérifier les journaux NAXSI pour confirmer le comportement attendu durant les tests.

## 8. Conclusion

Les mesures initiales indiquent un temps moyen de 7,1 ms pour le backend direct et de 10,3 ms via NGINX + NAXSI, soit un écart observé d'environ 3,2 ms.

Ces résultats fournissent une première référence pour le laboratoire. Ils ne suffisent pas à établir un surcoût universel du WAF ni à attribuer toute la différence au seul module NAXSI.
