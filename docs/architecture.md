# Architecture du laboratoire

## Architecture actuelle — Phase 1

```text
┌──────────────────────┐
│      Kali Linux      │
│   Client de test     │
└──────────┬───────────┘
           │ HTTP : 80
           ▼
┌──────────────────────┐
│        NGINX         │
│    Reverse Proxy     │
│  192.168.195.146     │
└──────────┬───────────┘
           │ HTTP : 3000
           ▼
┌──────────────────────┐
│    Python HTTP       │
│   Backend temporaire │
└──────────────────────┘
```

## Architecture cible

Kali Linux → NGINX + NAXSI → Application web de test.

## Évolution prévue

* Intégration du WAF.
* Ajout de règles de sécurité.
* Tests de détection et de blocage.
* Centralisation et analyse des journaux.
* Mesure de l'impact sur les performances.
