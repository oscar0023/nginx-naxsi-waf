# Phase 6 — Journalisation et analyse

## 1. Objectifs

Cette phase consiste à vérifier le fonctionnement de la journalisation de NGINX et NAXSI, analyser les événements de sécurité et distinguer les requêtes normales des requêtes bloquées.

Les objectifs sont les suivants :

- Vérifier la présence et le fonctionnement des journaux.
- Identifier les événements générés par NAXSI.
- Corréler les réponses HTTP avec les détections du WAF.
- Analyser les catégories de règles déclenchées.
- Rechercher d’éventuels faux positifs.
- Préparer l’évaluation des performances du WAF.

## 2. Environnement du laboratoire

| Élément | Configuration |
|---|---|
| Machine cliente | Kali Linux |
| Adresse IP cliente | `192.168.195.128` |
| Serveur WAF | Ubuntu Server |
| Nom du serveur | `waf-nginx-naxsi` |
| Adresse IP du WAF | `192.168.195.146` |
| Serveur web applicatif | DVWA |
| Adresse du backend | `127.0.0.1:3001` |
| Journal des accès | `/var/log/nginx/access.log` |
| Journal des erreurs | `/var/log/nginx/error.log` |

Les tests ont été réalisés dans un environnement de laboratoire contrôlé.

## 3. Vérification des journaux

Les journaux NGINX ont été vérifiés avec la commande suivante :

```bash
sudo ls -lh /var/log/nginx/
```

Les fichiers `access.log` et `error.log` étaient présents et contenaient des données.

Pour consulter les dernières requêtes HTTP :

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

Pour examiner les événements de sécurité :

```bash
sudo tail -n 20 /var/log/nginx/error.log
```

Le comptage des événements NAXSI a été réalisé avec :

```bash
sudo grep -c 'NAXSI_FMT' /var/log/nginx/error.log
```

**Résultat observé : 39 occurrences de `NAXSI_FMT`.**

Ce nombre représente les occurrences présentes dans le journal au moment de la vérification. Il ne correspond pas nécessairement à 39 attaques distinctes, car les tests répétés et les détections multiples peuvent augmenter ce total.

## 4. Analyse du journal d’accès

Le fichier `access.log` permet notamment d'identifier :

- L'adresse IP du client.
- La date et l'heure de la requête.
- La méthode HTTP et l'URI demandée.
- Le statut HTTP retourné.
- La taille de la réponse.
- L'agent utilisateur.

Les résultats observés comprennent les cas suivants :

| Requête | Statut HTTP | Interprétation |
|---|---:|---|
| `/login.php` | `200` | La page de connexion DVWA est accessible. |
| `/vulnerabilities/fi/?page=file1.php` | `200` | La requête reçoit une réponse normale de l'application. |
| `/` | `302` | Redirection applicative vers le parcours de connexion. |
| `/?q=bonjour` | `302` | Redirection applicative. |
| `/?id=123` | `302` | Redirection applicative. |
| Requête de traversée de répertoires | `403` | Requête refusée ; événements NAXSI correspondants présents. |
| `/?url=http://example.com` | `403` | Tentative de type RFI refusée et journalisée. |
| `/?id=%25U` | `403` | Séquence encodée détectée et requête refusée. |

Un statut `403` ne suffit pas, à lui seul, à prouver que NAXSI a bloqué une requête. Il faut le corréler avec un événement de sécurité correspondant dans `error.log`.

De même, un statut `302` indique une redirection, mais ne signifie pas que NAXSI a bloqué la requête.

## 5. Analyse des événements NAXSI

Les événements NAXSI sont enregistrés dans `error.log` sous la forme `NAXSI_FMT`.

Les champs importants sont :

| Champ | Signification |
|---|---|
| `ip` | Adresse IP du client |
| `uri` | URI concernée |
| `config=block` | Décision de blocage indiquée dans l'événement |
| `cscoreN` | Catégorie de détection |
| `scoreN` | Score associé à la catégorie |
| `zoneN` | Zone examinée, par exemple `ARGS` ou `URL` |
| `idN` | Identifiant de la règle déclenchée |
| `var_nameN` | Nom du paramètre concerné, lorsqu'il est disponible |
| `request` | Requête HTTP associée à l'événement |

Ces informations permettent de comprendre pourquoi une requête a été détectée et de faciliter l'analyse d'incidents.

## 6. Corrélation avec les tests de sécurité

Les événements enregistrés pendant la phase 5 ont été retrouvés dans les journaux.

| Test effectué | Catégorie observée | Règle | Score observé |
|---|---|---:|---:|
| Injection SQL (SQLi) | `$SQL` et `$XSS` | `1009`, `1013` | `22`, `40` |
| XSS réfléchi | `$XSS` | `1010` | `8` |
| Traversée de répertoires / tentative LFI | `$TRAVERSAL` | `1200` | `8` ou `16` |
| Tentative de type RFI | `$RFI` | `1100` | `8` |
| Séquence encodée `%U` | `$EVADE` | `1401` | `4` |
| Séquence à double encodage testée | `$XSS` | `1315` | `24` |

Les scores dépendent de la requête exacte. Les valeurs indiquées sont celles observées dans les événements disponibles.

### Observations

**Injection SQL :** les règles `1009` et `1013` ont été déclenchées. Les scores SQL et XSS étaient respectivement de 22 et 40 dans l'événement examiné.

**XSS réfléchi :** la règle `1010` a été déclenchée avec un score XSS de 8. La requête a reçu un statut HTTP `403`.

**Traversée de répertoires :** la règle `1200` a été déclenchée dans la zone `ARGS`, notamment sur le paramètre `page`. Les scores observés étaient de 8 ou 16 selon la variante testée.

**Tentative de type RFI :** la règle `1100` a été déclenchée sur le paramètre `url`, avec un score RFI de 8.

**Encodage suspect :** la séquence `%25U` a déclenché la règle `1401`, associée à la catégorie `$EVADE`, avec un score de 4.

**Double encodage :** la requête `/?id=%252e%252e%252f` a déclenché la règle `1315`, classée `$XSS` dans le journal, avec un score de 24. Elle ne doit donc pas être présentée comme une détection `$EVADE`.

Ces résultats confirment que les signatures testées ont été détectées. Ils ne démontrent pas que toutes les variantes possibles de ces attaques seraient détectées.

## 7. Vérification des requêtes ordinaires

Trois requêtes ordinaires ont été exécutées depuis Kali Linux.

### Test 1 — Page d'accueil

```bash
curl -sS -o /dev/null -w 'Accueil : HTTP %{http_code}\n' \
  'http://192.168.195.146/'
```

Résultat :

```text
Accueil : HTTP 302
```

### Test 2 — Paramètre texte

```bash
curl -sS -o /dev/null -w 'Texte : HTTP %{http_code}\n' \
  --get --data-urlencode 'q=bonjour' \
  'http://192.168.195.146/'
```

Résultat :

```text
Texte : HTTP 302
```

### Test 3 — Paramètre numérique

```bash
curl -sS -o /dev/null -w 'Numérique : HTTP %{http_code}\n' \
  --get --data-urlencode 'id=123' \
  'http://192.168.195.146/'
```

Résultat :

```text
Numérique : HTTP 302
```

Les trois requêtes apparaissent dans `access.log`. L'accès ultérieur à `/login.php`, avec un statut `200`, confirme que les redirections observées sont cohérentes avec le parcours d'authentification de DVWA.

Aucun événement de blocage NAXSI correspondant à ces trois requêtes ordinaires n'a été mis en évidence dans les extraits examinés.

**Conclusion sur les faux positifs :** aucun faux positif n'a été observé sur cet échantillon limité. Cette observation ne permet pas de garantir l'absence de faux positifs sur l'ensemble de l'application.

## 8. Commandes utiles pour poursuivre l'analyse

### 8.1 Compter les statuts HTTP

```bash
sudo awk '{count[$9]++} END {
    for (status in count)
        printf "%-6s %d\n", status, count[status]
}' /var/log/nginx/access.log
```

### 8.2 Compter les événements indiquant un blocage

```bash
sudo grep 'NAXSI_FMT' /var/log/nginx/error.log \
  | grep -c 'config=block'
```

### 8.3 Afficher les catégories de détection

```bash
sudo grep 'NAXSI_FMT' /var/log/nginx/error.log \
  | grep -oE 'cscore[0-9]+\=\$[A-Z]+' \
  | sort | uniq -c | sort -nr
```

Ces commandes fournissent des statistiques sur les journaux courants. Elles ne dédupliquent pas nécessairement les requêtes et ne permettent donc pas de calculer directement le nombre d'attaques distinctes.

## 9. Limites de l'analyse

Plusieurs limites doivent être prises en compte :

- Les événements analysés proviennent d'une période de test limitée.
- Les tests répétés peuvent apparaître plusieurs fois dans les journaux.
- Un statut HTTP isolé ne suffit pas à attribuer une décision au WAF.
- Les requêtes ordinaires testées sont peu nombreuses.
- Les résultats confirment le comportement observé sur les exemples testés, mais ne constituent pas une preuve de protection exhaustive.
- Une analyse plus complète des faux positifs nécessiterait des tests fonctionnels supplémentaires sur les différentes pages et fonctionnalités de DVWA.

## 10. Conclusion de la phase 6

La phase 6 a permis de vérifier la disponibilité des journaux NGINX, d'identifier les événements `NAXSI_FMT` et de corréler plusieurs requêtes refusées avec les catégories et règles de détection du WAF.

Les tests ont permis d'observer des détections SQLi, XSS, traversée de répertoires, RFI et EVADE. Les requêtes ordinaires examinées ont reçu des redirections applicatives `302`, cohérentes avec l'authentification DVWA, sans blocage NAXSI correspondant identifié dans les extraits consultés.

La journalisation et l'analyse de base sont donc opérationnelles dans le laboratoire. Les résultats doivent toutefois être interprétés dans les limites du nombre de tests réalisés.
