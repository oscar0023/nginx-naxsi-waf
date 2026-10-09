# Phase 4 — Déploiement de DVWA derrière NGINX et NAXSI

## Objectif

Déployer l'application volontairement vulnérable DVWA derrière le reverse proxy NGINX protégé par le WAF NAXSI.

## Architecture

- Client de test : Kali Linux (`192.168.195.128`)
- Serveur WAF : Ubuntu Server (`192.168.195.146`)
- Reverse proxy : NGINX, port 80
- WAF : NAXSI
- Application vulnérable : DVWA via Apache sur `127.0.0.1:3001`

## Résultats

Les journaux NGINX confirment le blocage des requêtes de test correspondant aux catégories SQLi, XSS, RFI et traversée de répertoires.

Les événements NAXSI contiennent `config=block`, les scores de détection et les identifiants des règles déclenchées.

Le backend Apache écoute sur l'interface locale `127.0.0.1:3001`, afin que les clients externes passent par NGINX.

## Conclusion

La chaîne NGINX → NAXSI → DVWA est configurée. Les journaux démontrent que NAXSI bloque plusieurs requêtes de test. Les contrôles finaux d'accessibilité et du niveau de sécurité DVWA doivent être conservés comme preuves complémentaires.
