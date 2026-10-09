# Phase 2 — Intégration et validation du WAF NGINX + NAXSI

## 1. Objectif

Cette phase intègre NAXSI à NGINX afin d'inspecter les requêtes HTTP entrantes et de bloquer celles qui atteignent les seuils configurés. Elle prolonge la phase 1, consacrée à la mise en place de NGINX comme reverse proxy et à la validation du backend.

Objectifs :

- compiler et charger le module NAXSI compatible avec NGINX ;
- charger les règles principales ;
- configurer les seuils de détection et de blocage ;
- tester des requêtes légitimes et des requêtes de laboratoire SQLi/XSS ;
- examiner les journaux et conserver des preuves reproductibles.

> Les essais sont limités au laboratoire contrôlé. Ne pas exécuter ces tests sur un système sans autorisation explicite.

## 2. Environnement du laboratoire

| Élément | Valeur observée |
|---|---|
| Client de test | Kali Linux — `192.168.195.128` |
| Serveur WAF | Ubuntu Server 22.04.5 LTS — `192.168.195.146` |
| NGINX | `1.18.0-6ubuntu14.21` |
| NAXSI | Version 1.7, compilée comme module dynamique |
| Module | `/usr/lib/nginx/modules/ngx_http_naxsi_module.so` |
| Règles principales | `/etc/nginx/naxsi/naxsi_core.rules` |
| Backend | Python `http.server` sur `127.0.0.1:3000` |
| Journal d'erreurs | `/var/log/nginx/error.log` |
| Journal d'accès | `/var/log/nginx/access.log` |

Ces valeurs décrivent l'environnement utilisé pendant les tests. Les commandes de vérification ci-dessous permettent de confirmer l'état réel du serveur au moment de reproduire le laboratoire.

## 3. Architecture

```text
Kali Linux (192.168.195.128)
          |
          | HTTP :80
          v
Ubuntu Server (192.168.195.146)
  NGINX + module NAXSI
          |
          | proxy_pass (requêtes autorisées)
          v
Backend Python (127.0.0.1:3000)
```

Le client envoie ses requêtes à NGINX. NAXSI inspecte la requête selon les règles chargées. Si un seuil de blocage est atteint, la requête est refusée. Sinon, NGINX la transmet au backend local. Le backend écoute sur `127.0.0.1`, et non sur toutes les interfaces réseau.

## 4. Installation et chargement de NAXSI

Le module a été compilé comme module dynamique compatible avec la version de NGINX installée sur Ubuntu. Les emplacements utilisés sont :

- `/usr/lib/nginx/modules/ngx_http_naxsi_module.so`
- `/etc/nginx/naxsi/naxsi_core.rules`

Dans `/etc/nginx/nginx.conf`, le module doit être chargé au niveau principal, avant les blocs `events` et `http` :

```nginx
load_module /usr/lib/nginx/modules/ngx_http_naxsi_module.so;
```

Dans le bloc `http`, les règles principales sont incluses :

```nginx
include /etc/nginx/naxsi/naxsi_core.rules;
```

Vérifications :

```bash
sudo nginx -t
sudo nginx -V 2>&1
ls -l /usr/lib/nginx/modules/ngx_http_naxsi_module.so
```

`nginx -t` valide la syntaxe et la cohérence de la configuration. Il ne remplace pas les tests fonctionnels du WAF.

## 5. Configuration de protection

Les seuils utilisés dans le laboratoire sont les suivants :

```nginx
location / {
    SecRulesEnabled;
    # LearningMode;  # désactivé pour les tests de blocage
    DeniedUrl "/RequestDenied";

    CheckRule "$SQL >= 8" BLOCK;
    CheckRule "$XSS >= 8" BLOCK;
    CheckRule "$RFI >= 8" BLOCK;
    CheckRule "$TRAVERSAL >= 5" BLOCK;
    CheckRule "$EVADE >= 4" BLOCK;

    proxy_pass http://127.0.0.1:3000;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

location = /RequestDenied {
    internal;
    return 403;
}
```

Cet extrait documente la configuration utilisée dans le laboratoire. Avant de le recopier, vérifier le fichier réellement activé sous `/etc/nginx/sites-enabled/` et confirmer que les directives correspondent à la version installée.

Le mode apprentissage sert à observer les détections et à préparer les règles. Il doit être désactivé pour vérifier le blocage effectif. Après toute modification :

```bash
sudo nginx -t && sudo systemctl reload nginx
```

## 6. Backend de test

Le backend Python est géré par le service systemd `web-test`. Le service utilise le compte `waf_server`, le répertoire `/home/waf_server/web-test` et écoute sur `127.0.0.1:3000`.

Extrait du service `/etc/systemd/system/web-test.service` :

```ini
[Unit]
Description=Backend Web de test pour NGINX et NAXSI
After=network.target

[Service]
Type=simple
User=waf_server
WorkingDirectory=/home/waf_server/web-test
ExecStart=/usr/bin/python3 -m http.server 3000 --bind 127.0.0.1
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Commandes de contrôle :

```bash
sudo systemctl status web-test --no-pager
sudo ss -lntp | grep ':3000'
curl -i http://127.0.0.1:3000/
```

La réponse du backend contient le texte `Backend Web operationnel`.

## 7. Protocole de test

Exécuter les tests depuis Kali Linux ou depuis une machine de laboratoire autorisée pouvant joindre `192.168.195.146`.

### 7.1 Requête légitime

```bash
curl -i http://192.168.195.146/
```

**Résultat observé :** `HTTP/1.1 200 OK`, avec le contenu `Backend Web operationnel`. Cela confirme que NGINX transmet une requête normale au backend.

### 7.2 SQLi en mode apprentissage

```bash
curl -i 'http://192.168.195.146/?id=1%20UNION%20SELECT%201'
```

Pour examiner les événements :

```bash
sudo grep -Ei 'NAXSI_FMT|naxsi' /var/log/nginx/error.log | tail -n 20
```

L'événement observé indiquait la catégorie `$SQL`, un score de `8`, la zone `ARGS`, l'identifiant de règle `1000` et la variable `id`. En mode apprentissage, l'événement est journalisé sans que le blocage soit nécessairement appliqué.

### 7.3 SQLi en mode blocage

Après désactivation de `LearningMode`, valider puis recharger NGINX :

```bash
sudo nginx -t && sudo systemctl reload nginx
```

Répéter la requête :

```bash
curl -i 'http://192.168.195.146/?id=1%20UNION%20SELECT%201'
```

**Résultat observé :** `HTTP/1.1 403 Forbidden`. Le journal indique le mode `block`, un score SQL de `8` et la règle `1000`. Le blocage SQLi est donc confirmé.

### 7.4 XSS

```bash
curl -i 'http://192.168.195.146/?q=%3Cscript%3Ealert(1)%3C%2Fscript%3E'
```

Le journal observé signalait la catégorie `$XSS`, un score de `8`, la zone `ARGS`, la règle `1010` et la variable `q`. Cela confirme la détection. Pour confirmer séparément le blocage, conserver aussi le code HTTP réellement renvoyé par `curl` et l'événement correspondant du journal.

## 8. Résultats

| Scénario | Résultat observé | Conclusion |
|---|---|---|
| Requête légitime | `200 OK` | Requête transmise au backend |
| SQLi en mode apprentissage | `$SQL`, score `8`, règle `1000` | Détection journalisée |
| SQLi en mode blocage | `403 Forbidden`, mode `block` | Blocage confirmé |
| Test XSS | `$XSS`, score `8`, règle `1010` | Détection confirmée ; consigner le code HTTP pour établir le blocage |
| Backend local | `200 OK` sur `127.0.0.1:3000` | Backend opérationnel |
| Port backend | écoute sur `127.0.0.1:3000` | Backend non exposé directement sur toutes les interfaces |

## 9. Journaux et diagnostic

```bash
sudo tail -n 50 /var/log/nginx/error.log
sudo tail -n 50 /var/log/nginx/access.log
sudo grep -Ei 'NAXSI_FMT|naxsi' /var/log/nginx/error.log | tail -n 20
```

Pour interpréter correctement un événement, rapprocher l'heure, la requête, le score, la zone, l'identifiant de règle et le code HTTP reçu par le client.

## 10. Limites

- Ces tests couvrent quelques scénarios contrôlés et ne prouvent pas une protection complète contre toutes les attaques Web.
- Les seuils de score sont propres à ce laboratoire ; une application réelle nécessite des tests et une adaptation.
- Les faux positifs doivent être étudiés avant un déploiement en production.
- Le backend Python est un serveur de test minimal, pas un serveur de production.
- Le laboratoire utilise HTTP ; TLS n'est pas encore configuré.
- Les captures d'écran sont des preuves complémentaires : la configuration et les commandes reproductibles restent indispensables.

## 11. Checklist de clôture

- [ ] `sudo nginx -t` réussit.
- [ ] Une requête légitime renvoie `200 OK`.
- [ ] La détection SQLi est visible dans les journaux.
- [ ] Le blocage SQLi renvoie `403 Forbidden`.
- [ ] La détection XSS est visible dans les journaux.
- [ ] Le code HTTP du test XSS est vérifié et documenté.
- [ ] Le service backend est actif et lié à `127.0.0.1:3000`.
- [ ] Les captures sont enregistrées dans `screenshots/`.
- [ ] `screenshots/README.md` décrit chaque capture.
- [ ] Aucun secret ni donnée sensible n'est publié.

## 12. Conclusion

La phase 2 a permis d'intégrer NAXSI à NGINX et de valider la détection de requêtes de test SQLi et XSS. Le blocage SQLi a été confirmé par une réponse `403 Forbidden`, tandis qu'une requête légitime atteint le backend. Pour le test XSS, la documentation doit distinguer la détection dans le journal du blocage effectif et inclure le code HTTP observé.

La phase suivante portera sur la compréhension des règles NAXSI : identifiants, zones inspectées, scores et logique de décision.
