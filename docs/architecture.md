# Architecture du laboratoire — Phases 1 et 2

## 1. Évolution du projet

La phase 1 a établi le chemin entre le client de test, NGINX et un backend Web minimal. La phase 2 a ajouté le module NAXSI, chargé les règles principales et validé la détection et le blocage de requêtes de test.

| Phase | Travaux | État |
|---|---|---|
| Phase 1 — Préparation du laboratoire et NGINX | Kali Linux, Ubuntu Server, NGINX reverse proxy, backend Python et test HTTP de base | Terminée |
| Phase 2 — Intégration et validation du WAF NGINX + NAXSI | Module dynamique NAXSI, règles principales, seuils, tests SQLi/XSS et consultation des journaux | Tests principaux réalisés ; captures et documentation à finaliser |

## 2. Composants du laboratoire

| Composant | Adresse / chemin | Rôle |
|---|---|---|
| Client Kali Linux | `192.168.195.128` | Génère les requêtes de validation |
| Serveur Ubuntu | `192.168.195.146:80` | Point d'entrée HTTP du laboratoire |
| NGINX | Port `80` | Reverse proxy et application des directives de contrôle |
| Module NAXSI | `/usr/lib/nginx/modules/ngx_http_naxsi_module.so` | Inspecte les requêtes HTTP |
| Règles principales | `/etc/nginx/naxsi/naxsi_core.rules` | Règles NAXSI chargées par NGINX |
| Backend Python | `127.0.0.1:3000` | Application de test minimale |
| Journaux | `/var/log/nginx/error.log` et `/var/log/nginx/access.log` | Traces des détections, blocages et accès |

Versions observées : Ubuntu Server 22.04.5 LTS, NGINX `1.18.0-6ubuntu14.21` et NAXSI 1.7 compilé comme module dynamique.

## 3. Schéma de flux

```text
                 Client de test
            Kali Linux 192.168.195.128
                        |
                        | HTTP vers le port 80
                        v
        +----------------------------------+
        | Ubuntu Server 192.168.195.146    |
        |                                  |
        | NGINX + module NAXSI              |
        | - inspection des requêtes        |
        | - calcul des scores              |
        | - autorisation ou blocage        |
        +----------------+-----------------+
                         |
           Requête autorisée uniquement
                         |
                         | proxy_pass
                         v
              +-----------------------+
              | Backend Python        |
              | 127.0.0.1:3000         |
              | Application de test    |
              +-----------------------+
```

## 4. Traitement d'une requête

1. Le client envoie une requête HTTP à `192.168.195.146:80`.
2. NGINX reçoit la requête et applique les directives NAXSI configurées dans le site.
3. NAXSI inspecte les éléments de la requête selon les règles chargées et journalise les détections.
4. Si un seuil de blocage est atteint en mode protection, la requête est refusée. Dans les tests SQLi, la réponse observée était `403 Forbidden`.
5. Si la requête est autorisée, NGINX la transmet au backend sur `127.0.0.1:3000`.
6. Les journaux NGINX sont utilisés pour analyser les accès et les événements NAXSI.

## 5. Isolation du backend

Le service `web-test` est géré par systemd et lance le serveur Python avec :

```bash
/usr/bin/python3 -m http.server 3000 --bind 127.0.0.1
```

L'écoute sur `127.0.0.1` limite l'accès direct au backend depuis les autres machines. Les clients distants doivent passer par NGINX.

Commandes de vérification :

```bash
sudo systemctl status web-test --no-pager
sudo ss -lntp | grep ':3000'
curl -i http://127.0.0.1:3000/
curl -i http://192.168.195.146/
```

## 6. Intégration NAXSI

Le module dynamique est chargé dans la configuration principale de NGINX :

```nginx
load_module /usr/lib/nginx/modules/ngx_http_naxsi_module.so;
```

Les règles principales sont incluses dans le bloc `http` :

```nginx
include /etc/nginx/naxsi/naxsi_core.rules;
```

Seuils de blocage documentés dans le site :

```nginx
CheckRule "$SQL >= 8" BLOCK;
CheckRule "$XSS >= 8" BLOCK;
CheckRule "$RFI >= 8" BLOCK;
CheckRule "$TRAVERSAL >= 5" BLOCK;
CheckRule "$EVADE >= 4" BLOCK;
```

Vérifier la configuration réellement active avant de reproduire les extraits :

```bash
sudo nginx -t
sudo nginx -T
```

## 7. Résultats validés

| Test | Résultat observé |
|---|---|
| Requête normale `GET /` | `200 OK` et contenu du backend |
| SQLi en mode apprentissage | `$SQL`, score `8`, zone `ARGS`, règle `1000` |
| SQLi en mode blocage | `403 Forbidden`, mode `block` |
| Test XSS | `$XSS`, score `8`, zone `ARGS`, règle `1010` |
| Backend local | `200 OK` sur `127.0.0.1:3000` |
| Écoute du backend | `127.0.0.1:3000` |

Pour le test XSS, consigner le code HTTP retourné par `curl` afin de distinguer la détection dans le journal du blocage effectif.

## 8. Phases suivantes

- **Phase 3 — Compréhension des règles NAXSI :** règles, identifiants, zones, scores et logique de décision.
- **Phase 4 — Mise en place d'une application vulnérable :** utiliser une application volontairement vulnérable dans un environnement isolé.
- **Phase 5 — Tests de sécurité Web :** élargir les scénarios contrôlés et comparer requêtes, réponses et journaux.
- **Phase 6 — Optimisation et gestion des faux positifs :** identifier les requêtes légitimes bloquées et ajuster les règles.
- **Phase 7 — Performance et rapport final :** mesurer l'impact, documenter les limites et présenter les résultats.

## 9. Limites

Cette architecture décrit un laboratoire pédagogique fonctionnant en HTTP. Elle ne prouve pas une couverture exhaustive des attaques Web ni une configuration prête pour la production. Un déploiement réel demanderait notamment TLS, une stratégie de mise à jour des règles, une analyse des faux positifs, de la supervision et des tests de performance.
