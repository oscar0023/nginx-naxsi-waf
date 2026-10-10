# Web Application Firewall avec NGINX & NAXSI

## Présentation

Ce projet consiste à concevoir, déployer et évaluer un Web Application Firewall (WAF) open source basé sur NGINX et NAXSI.

L'objectif est de protéger une application web contre plusieurs catégories d'attaques et d'étudier les mécanismes de détection, de blocage et de journalisation.

## Objectifs

* Déployer NGINX comme reverse proxy.
* Intégrer NAXSI.
* Configurer et personnaliser les règles de sécurité.
* Tester la détection des attaques web.
* Analyser les journaux de sécurité.
* Mesurer les performances et les faux positifs.

## Technologies

* Linux
* NGINX
* NAXSI
* OWASP Top 10
* OWASP ZAP
* GitHub

## Architecture

Kali Linux (tests) → NGINX + NAXSI (WAF) → Application web de test.

## Progression du projet

* [x] Phase 1 — Préparation du laboratoire et reverse proxy
* [x] Phase 2 — Installation et intégration de NAXSI
* [x] Phase 3 — Compréhension des règles NAXSI
* [x] Phase 4 — Déploiement de DVWA derrière NGINX et NAXSI
* [x] Phase 5 — Tests de sécurité web
* [x] Phase 6 — Journalisation et analyse
* [x] Phase 7 — Évaluation des performances
* [x] Phase 8 — Rapport final

## Documentation

* [Phase 1 — Préparation du laboratoire et reverse proxy](docs/phase-1.md)
* [Phase 2 — Installation et intégration de NAXSI](docs/phase-2.md)
* [Phase 3 — Compréhension des règles NAXSI](docs/phase-3.md)
* [Phase 4 — Déploiement de DVWA derrière NGINX et NAXSI](docs/phase-4.md)
* [Phase 5 — Tests de sécurité web](docs/phase-5.md)
* [Phase 6 — Journalisation et analyse](docs/phase-6.md)
* [Phase 7 — Évaluation des performances](docs/phase-7.md)
* [Phase 8 — Rapport final](rapport_waf.pdf)
* [Architecture du laboratoire](docs/architecture.md)

## Environnement de test

Les tests seront effectués exclusivement sur des systèmes de laboratoire autorisés.

## Auteur
### Oscar ALIDJINOU
Étudiant ingénieur en Sécurité IT & Confiance Numérique.

## Avertissement

Ce projet est destiné à l'apprentissage et à l'évaluation de la sécurité dans un environnement contrôlé.
