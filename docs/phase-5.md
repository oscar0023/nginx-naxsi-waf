# Phase 5 — Tests de sécurité 

## 1. Objectifs

Cette phase a pour objectif de vérifier le fonctionnement des règles de sécurité du Web Application Firewall (WAF) déployé avec NGINX et NAXSI.

Les tests ont été réalisés dans un environnement de laboratoire contrôlé, depuis une machine Kali Linux vers le serveur Ubuntu hébergeant le WAF et l'application de test DVWA.

Les objectifs sont les suivants :

- Vérifier la détection de plusieurs catégories d'attaques Web.
- Vérifier le blocage des requêtes correspondant aux seuils configurés.
- Examiner les journaux NGINX/NAXSI afin d'identifier les règles déclenchées.
- Constituer des preuves techniques pour le rapport du projet.

## 2. Environnement de test

| Composant | Configuration |
|---|---|
| Machine de test | Kali Linux |
| Adresse IP du client | `192.168.195.128` |
| Serveur WAF | Ubuntu Server 22.04.5 LTS |
| Adresse IP du WAF | `192.168.195.146` |
| Serveur Web frontal | NGINX 1.18.0 |
| Module WAF | NAXSI 1.7 |
| Application de test | DVWA |
| Serveur Web en arrière-plan | Apache sur `127.0.0.1:3001` |
| Journal étudié | `/var/log/nginx/error.log` |

Le trafic HTTP provenant du client passe par NGINX et NAXSI avant d'être transmis à l'application en arrière-plan.

## 3. Configuration des seuils de sécurité

Les seuils suivants sont configurés dans le bloc `location /` du site NGINX :

```nginx
CheckRule "$SQL >= 8" BLOCK;
CheckRule "$XSS >= 8" BLOCK;
CheckRule "$RFI >= 8" BLOCK;
CheckRule "$TRAVERSAL >= 5" BLOCK;
CheckRule "$EVADE >= 4" BLOCK;
```

Ces directives demandent à NAXSI de bloquer les requêtes lorsque le score cumulé de la catégorie concernée atteint le seuil défini.

Les règles de détection proviennent du fichier :

```text
/etc/nginx/naxsi/naxsi_core.rules
```

## 4. Méthodologie

Pour chaque catégorie, une requête de test a été envoyée depuis Kali Linux vers le serveur WAF.

Les vérifications reposent sur deux éléments complémentaires :

1. **La réponse HTTP :** un statut `403 Forbidden` indique que la requête a été refusée.
2. **Le journal NAXSI :** le champ `config=block`, la catégorie de score, l'identifiant de règle et la zone inspectée permettent d'identifier le mécanisme de détection.

Une réponse HTTP 403, à elle seule, ne suffit pas à prouver quelle règle a provoqué le blocage. L'analyse des journaux est donc indispensable.

## 5. Tests réalisés et résultats

### 5.1. Injection SQL (SQLi)

**Objectif :** vérifier que NAXSI détecte et bloque une tentative d'injection SQL.

Le test a été effectué sur la fonctionnalité SQL Injection de DVWA, avec une valeur de paramètre conçue pour tester une condition SQL manipulée.

Extrait représentatif du journal :

```text
config=block
cscore0=$SQL
score0=22
zone0=ARGS
id0=1009
var_name0=id
```

Une requête de test a également déclenché plusieurs règles, avec un score SQL de 22 et un score XSS de 40.

**Résultat :** test concluant. Le journal confirme le blocage et la détection d'un motif classé SQL.

### 5.2. Cross-Site Scripting (XSS)

**Objectif :** vérifier la détection d'une tentative de XSS réfléchi.

La requête a été envoyée sur la fonctionnalité XSS réfléchi de DVWA avec une valeur contenant une balise script.

Exemple de requête :

```http
GET /vulnerabilities/xss_r/?name=%3Cscript%3Ealert%28%27XSS%27%29%3C%2Fscript%3E
```

Extrait du journal :

```text
config=block
cscore0=$XSS
score0=8
zone0=ARGS
id0=1010
var_name0=name
```

**Résultat :** test concluant. NAXSI a détecté le motif XSS dans le paramètre `name` et a bloqué la requête.

### 5.3. Directory Traversal et inclusion de fichiers (LFI)

**Objectif :** vérifier que NAXSI détecte une tentative de navigation hors du répertoire prévu et d'accès à un fichier système.

Le test a été effectué sur la fonctionnalité File Inclusion de DVWA en utilisant une valeur de paramètre comportant des séquences de remontée de répertoires.

Extrait du journal :

```text
config=block
cscore0=$TRAVERSAL
score0=16
zone0=ARGS
id0=1200
var_name0=page
```

**Résultat :** test concluant. NAXSI a détecté le motif de traversal dans le paramètre `page` et a bloqué la requête.

Ce résultat démontre la détection du scénario testé ; il ne garantit pas la protection contre toutes les variantes de traversal ou d'inclusion de fichiers.

### 5.4. Remote File Inclusion (RFI)

**Objectif :** vérifier la détection d'une valeur de paramètre contenant une URL externe.

Requête de test :

```bash
curl -i "http://192.168.195.146/?url=http://example.com"
```

Extrait du journal :

```text
config=block
cscore0=$RFI
score0=8
zone0=ARGS
id0=1100
var_name0=url
```

**Résultat :** test concluant. Le journal confirme le déclenchement de la règle RFI `1100` et le blocage de la requête.

Ce test valide la détection de la signature utilisée, et non la couverture de toutes les formes possibles de RFI.

### 5.5. Techniques d'évasion (EVADE)

**Objectif :** vérifier qu'une signature associée à une technique d'évasion est détectée.

La règle `1401`, présente dans le fichier `naxsi_core.rules`, recherche la chaîne `%U` et attribue quatre points à la catégorie `$EVADE`.

Requête de test :

```bash
curl --path-as-is -i 'http://192.168.195.146/?id=%25U'
```

Extrait du journal :

```text
config=block
cscore0=$EVADE
score0=4
zone0=ARGS
id0=1401
var_name0=id
```

**Résultat :** test concluant. La règle `1401` a été déclenchée dans la zone `ARGS`. Le score EVADE atteint le seuil configuré de 4 points et la requête est bloquée.

**Observation importante :** un test précédent utilisant un double encodage avait déclenché la règle `1315`, qui attribue des points à `$XSS`. Ce résultat ne constituait pas une preuve de détection EVADE. La requête `%25U` a permis de confirmer spécifiquement cette catégorie.

## 6. Synthèse des résultats

| Catégorie | Seuil configuré | Preuve observée | Statut |
|---|---:|---|---|
| SQL Injection | 8 | Score SQL de 22 dans un test DVWA | Réussi |
| XSS | 8 | Score XSS de 8, règle `1010` | Réussi |
| Directory Traversal / LFI | 5 | Score TRAVERSAL de 16, règle `1200` | Réussi |
| RFI | 8 | Score RFI de 8, règle `1100` | Réussi |
| EVADE | 4 | Score EVADE de 4, règle `1401` | Réussi |

Les cinq catégories ont ainsi fait l'objet d'au moins un test concluant, étayé par les journaux NAXSI.


## 7. Limites des tests

Les tests réalisés valident le comportement des règles pour les requêtes et les signatures effectivement utilisées. Ils ne constituent pas un audit exhaustif de la sécurité de l'application.

Les points suivants restent à évaluer dans les phases suivantes :

- Les faux positifs sur les requêtes légitimes.
- Les variantes d'attaques non couvertes par les scénarios testés.
- L'impact du WAF sur les performances et le temps de réponse.
- La qualité de la journalisation et la facilité d'analyse des événements.
- La robustesse de la configuration dans des conditions de trafic plus variées.

Un statut HTTP 403 doit toujours être interprété avec son journal associé pour éviter d'attribuer à tort un blocage à NAXSI lorsqu'il provient d'un autre composant.

## 8. Conclusion 

La phase 5 a permis de vérifier le fonctionnement des mécanismes de détection et de blocage NAXSI pour cinq catégories : injection SQL, XSS, Directory Traversal/LFI, RFI et EVADE.

Les journaux confirment le déclenchement des règles attendues dans les scénarios retenus. Les seuils configurés ont été atteints et les requêtes de test ont été bloquées.

Cette phase constitue une validation fonctionnelle initiale du WAF. Elle ne permet pas de conclure à une protection complète contre toutes les attaques Web.

La suite du projet portera sur les tests des requêtes légitimes, l'analyse des faux positifs et l'évaluation de l'impact du WAF sur les performances.
