# Phase 1 — Préparation du laboratoire

## 1. Objectif

Mettre en place un laboratoire de sécurité web dans lequel NGINX agit comme reverse proxy devant un backend HTTP.

## 2. Environnement

* Kali Linux : machine utilisée pour effectuer les tests.
* Ubuntu Server : machine hébergeant NGINX.
* NGINX : reverse proxy sur le port 80.
* Python HTTP Server : backend temporaire sur le port 3000.

## 3. Architecture déployée

Kali Linux → NGINX → Backend Python.

Le client Kali envoie une requête HTTP à l'adresse du serveur NGINX. NGINX transmet ensuite la requête au backend Python.

## 4. Configuration réseau

* Adresse du serveur NGINX : `192.168.195.146`
* Port d'entrée HTTP : `80`
* Port du backend : `3000`

## 5. Travaux réalisés

* Vérification de la connectivité entre Kali et Ubuntu.
* Installation et vérification de NGINX.
* Configuration du reverse proxy.
* Démarrage du backend HTTP Python.
* Test de l'accès au service depuis Kali.
* Vérification de la réponse HTTP 200 OK.
* Observation des journaux d'accès NGINX.

## 6. Résultat

Le reverse proxy fonctionne : les requêtes envoyées depuis Kali à NGINX sont transmises au backend Python.

## 7. Limites

Le serveur Python est utilisé uniquement comme backend temporaire de test. Il ne constitue pas une application de production.

NAXSI n'est pas encore intégré. La détection et le blocage des attaques seront évalués dans les phases suivantes.

## 8. Prochaine étape

Installer et intégrer NAXSI à NGINX, puis vérifier son fonctionnement à l'aide de requêtes de test contrôlées.
