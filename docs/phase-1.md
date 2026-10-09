# Phase 1 — Préparation du laboratoire

## 1. Objectif

Mettre en place un laboratoire de sécurité web avec NGINX utilisé comme reverse proxy devant un backend HTTP.

## 2. Environnement technique

* Kali Linux : machine de test.
* Ubuntu Server : serveur WAF.
* NGINX : reverse proxy HTTP.
* Python HTTP Server : backend temporaire de test.

## 3. Architecture

Kali Linux → NGINX (port 80) → Backend Python (port 3000).

## 4. Travaux réalisés

* Installation et vérification de NGINX.
* Configuration du reverse proxy.
* Démarrage du backend HTTP.
* Test de connectivité depuis Kali Linux.
* Vérification de la réponse HTTP 200 OK.
* Consultation des journaux NGINX.

## 5. Résultats

Le reverse proxy transmet correctement les requêtes HTTP vers le backend de test.

## 6. Limites

Le backend Python est temporaire et ne constitue pas une application de production. NAXSI n'est pas encore intégré.

## 7. Prochaine étape

Installer et intégrer NAXSI afin de détecter et bloquer les requêtes web malveillantes dans un laboratoire contrôlé.
