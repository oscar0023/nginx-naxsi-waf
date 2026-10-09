# Phase 3 — Compréhension des règles NAXSI

## 1. Objectifs

Cette phase a pour objectifs de :

- Comprendre le fonctionnement des règles principales de NAXSI.
- Étudier les scores attribués aux requêtes suspectes.
- Comprendre les seuils de blocage configurés dans NGINX.
- Tester plusieurs catégories d'attaques Web dans un environnement contrôlé.
- Vérifier que les requêtes légitimes continuent de fonctionner.
- Analyser les événements générés dans les journaux NGINX.

## 2. Architecture du laboratoire

| Composant | Configuration |
|---|---|
| Machine cliente | Kali Linux |
| IP du client | `192.168.195.128` |
| Serveur WAF | Ubuntu Server 22.04.5 LTS |
| Nom du serveur | `waf-nginx-naxsi` |
| IP du serveur WAF | `192.168.195.146` |
| Serveur Web frontal | NGINX `1.18.0-6ubuntu14.21` |
| Module de sécurité | NAXSI |
| Fichier des règles principales | `/etc/nginx/naxsi/naxsi_core.rules` |
| Configuration du site | `/etc/nginx/sites-available/waf` |
| Journal des événements | `/var/log/nginx/error.log` |
| Backend de test | Python HTTP Server sur `127.0.0.1:3000` |

### Schéma logique

```text
Kali Linux
192.168.195.128
       |
       | Requêtes HTTP
       v
NGINX + NAXSI
192.168.195.146:80
       |
       | Requêtes autorisées
       v
Backend Python
127.0.0.1:3000
```

NGINX constitue le point d'entrée du laboratoire. NAXSI inspecte les requêtes HTTP et applique les règles de sécurité avant leur transmission au backend.

## 3. Configuration des seuils de blocage

Le bloc `location /` de la configuration NGINX contient les directives suivantes :

```nginx
SecRulesEnabled;
# LearningMode;
DeniedUrl "/RequestDenied";

CheckRule "$SQL >= 8" BLOCK;
CheckRule "$XSS >= 8" BLOCK;
CheckRule "$RFI >= 8" BLOCK;
CheckRule "$TRAVERSAL >= 5" BLOCK;
CheckRule "$EVADE >= 4" BLOCK;
```

Le mode apprentissage est désactivé pour les tests de blocage.

Les directives `CheckRule` définissent les seuils utilisés pour décider si une requête doit être bloquée en fonction du score associé à une catégorie.

| Catégorie | Seuil de blocage |
|---|---:|
| SQL Injection | 8 |
| Cross-Site Scripting (XSS) | 8 |
| Remote File Inclusion (RFI) | 8 |
| Directory Traversal | 5 |
| Évasion de règles | 4 |

La directive `DeniedUrl` est configurée vers `/RequestDenied`, un emplacement interne qui renvoie une réponse HTTP 403.

## 4. Étude des règles principales

Les règles principales de NAXSI sont définies dans :

`/etc/nginx/naxsi/naxsi_core.rules`

### 4.1 SQL Injection

La règle `1000` recherche notamment des mots-clés SQL tels que `select`, `union`, `update`, `delete`, `insert` et `drop`.

```nginx
MainRule "rx:select|union|update|delete|insert|table|from|ascii|hex|unhex|drop|load_file|substr|group_concat|dumpfile|bigint" "msg:sql keywords" "mz:BODY|URL|ARGS|$HEADERS_VAR:Cookie" "s:$SQL:4" id:1000;
```

Cette règle attribue 4 points au score SQL lorsqu'elle correspond. Le score enregistré dépend des correspondances effectivement constatées.

### 4.2 Remote File Inclusion (RFI)

La règle `1100` recherche le motif `http://` dans les zones configurées :

```nginx
MainRule "str:http://" "msg:http:// scheme" "mz:ARGS|BODY|$HEADERS_VAR:Cookie" "s:$RFI:8" id:1100;
```

Cette règle attribue 8 points au score RFI. Dans notre configuration, ce score atteint le seuil de blocage.

### 4.3 Directory Traversal

La règle `1200` détecte la séquence `..` :

```nginx
MainRule "str:.." "msg:double dot" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$TRAVERSAL:4" id:1200;
```

La règle attribue 4 points au score de traversal lorsqu'elle correspond. Plusieurs correspondances peuvent contribuer au score enregistré.

### 4.4 Cross-Site Scripting (XSS)

La règle `1010` détecte une parenthèse ouvrante et contribue aux scores SQL et XSS :

```nginx
MainRule "str:(" "msg:open parenthesis, probable sql/xss" "mz:ARGS|URL|BODY|$HEADERS_VAR:Cookie" "s:$SQL:4,$XSS:8" id:1010;
```

D'autres règles détectent notamment les caractères `<` et `>` :

- `1302` : balise HTML ouvrante potentielle.
- `1303` : balise HTML fermante potentielle.

Les règles fondées sur des motifs ne prouvent pas à elles seules qu'une attaque est exploitable. Certaines requêtes légitimes peuvent contenir ces caractères.

### 4.5 Évasion

Les règles `1400` et `1401` recherchent respectivement les motifs `&#` et `%U`, susceptibles d'indiquer certains mécanismes d'encodage.

Elles contribuent au score `$EVADE`, dont le seuil de blocage est fixé à 4.

## 5. Tests réalisés

Tous les tests ont été exécutés dans le laboratoire autorisé, depuis Kali Linux vers `192.168.195.146`.

### 5.1 Test SQL Injection

Commande :

```bash
curl -i 'http://192.168.195.146/?id=1%20UNION%20SELECT%201'
```

**Résultat : HTTP 403 Forbidden.**

Événement observé dans le journal :

- `config=block`
- `score0=8`
- `zone0=ARGS`
- `id0=1000`
- `var_name0=id`

Interprétation : la règle SQL `1000` a été déclenchée sur le paramètre `id`. Le score SQL enregistré atteint le seuil de blocage configuré.

### 5.2 Test XSS

Commande :

```bash
curl -i 'http://192.168.195.146/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E'
```

**Résultat : HTTP 403 Forbidden.**

Événement observé :

- `config=block`
- `score0=4` pour `$SQL`
- `score1=8` pour `$XSS`
- `zone0=ARGS`
- `id0=1010`
- `var_name0=q`

Interprétation : la règle `1010` a contribué aux scores SQL et XSS. Le score XSS enregistré atteint le seuil de blocage de 8.

### 5.3 Test RFI

Commande :

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' \
'http://192.168.195.146/?url=http%3A%2F%2Fexample.com'
```

**Résultat : HTTP 403.**

Événement observé :

- `config=block`
- `cscore0=$RFI`
- `score0=8`
- `id0=1100`
- `var_name0=url`

Interprétation : la règle `1100` a détecté le motif `http://` dans le paramètre `url`. Le score RFI atteint le seuil configuré.

### 5.4 Test Directory Traversal

Commande :

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' \
'http://192.168.195.146/?file=../../etc/passwd'
```

**Résultat : HTTP 403.**

Événement observé :

- `config=block`
- `cscore0=$TRAVERSAL`
- `score0=8`
- `id0=1200`
- `var_name0=file`

Interprétation : la règle `1200` a détecté la séquence `..` dans le paramètre `file`. Le score enregistré de 8 dépasse le seuil de traversal fixé à 5.

### 5.5 Test d'une requête légitime

Commande :

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' \
'http://192.168.195.146/?page=accueil'
```

**Résultat : HTTP 200.**

Interprétation : la requête de référence a reçu une réponse réussie. Ce résultat indique que cette requête légitime n'a pas été bloquée.

## 6. Synthèse des résultats

| Test | Code HTTP | Règle observée dans le journal |
|---|---:|---|
| SQL Injection | 403 | `1000` |
| XSS | 403 | `1010` |
| RFI | 403 | `1100` |
| Directory Traversal | 403 | `1200` |
| Requête légitime | 200 | Aucun événement correspondant fourni dans l'extrait étudié |

Les quatre requêtes de test présentant des motifs suspects ont été bloquées. La requête légitime de référence a reçu une réponse HTTP 200.

Ces résultats confirment le fonctionnement attendu du WAF pour les cas testés, mais ne constituent pas une preuve de protection exhaustive contre toutes les variantes d'attaque.

## 7. Commandes de vérification

Vérifier la configuration NGINX :

```bash
sudo nginx -t
```

Afficher les événements NAXSI :

```bash
sudo grep 'NAXSI_FMT' /var/log/nginx/error.log | tail -n 20
```

Rechercher les principales règles étudiées :

```bash
sudo grep -nE 'id:1000|id:1010|id:1100|id:1200|id:1302|id:1303' \
/etc/nginx/naxsi/naxsi_core.rules
```

Vérifier le backend :

```bash
curl -i http://127.0.0.1:3000/
sudo systemctl status web-test --no-pager
```

## 8. Limites et précautions

- Les tests ont été réalisés dans un environnement de laboratoire contrôlé.
- Les résultats concernent uniquement les requêtes effectivement testées.
- Les règles basées sur des motifs peuvent produire des faux positifs.
- Un code HTTP 403 confirme le refus de la requête ; les journaux permettent d'identifier la catégorie, le score et la règle rapportés.
- Le backend Python est un serveur de test simple et ne représente pas à lui seul une application Web vulnérable réaliste.
- La validation de vulnérabilités applicatives nécessitera une application de test dédiée lors de la phase 4.

## 9. Captures d'écran

Les captures attendues pour cette phase sont répertoriées dans `screenshots/README.md`.

Les captures doivent présenter des résultats réellement obtenus et rester lisibles. Aucune capture ne doit être fabriquée ou présentée comme une preuve si le test correspondant n'a pas été exécuté.

## 10. Conclusion

Cette phase a permis d'étudier les règles principales de NAXSI, de comprendre les scores par catégorie et d'observer le blocage de tests SQL Injection, XSS, RFI et Directory Traversal.

La requête légitime de référence a également été acceptée. Les codes HTTP et les événements `NAXSI_FMT` constituent les principales preuves recueillies.

Le laboratoire est prêt pour la phase 4, consacrée à l'intégration d'une application volontairement vulnérable derrière NGINX et NAXSI.
