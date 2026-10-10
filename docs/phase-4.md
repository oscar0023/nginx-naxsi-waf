# Phase 4 — Déploiement de DVWA derrière NGINX et NAXSI

## 1. Objectifs

Cette phase consiste à déployer l'application web volontairement vulnérable **Damn Vulnerable Web Application (DVWA)** derrière un reverse proxy NGINX intégrant le Web Application Firewall NAXSI.

Les objectifs sont les suivants :

- Déployer DVWA sur le serveur Ubuntu.
- Configurer Apache et MariaDB pour faire fonctionner l'application.
- Faire transiter les requêtes clientes par NGINX et NAXSI.
- Limiter l'accès direct au serveur web applicatif.
- Vérifier la détection et le blocage de requêtes web suspectes.
- Collecter des preuves techniques pour le rapport final.

## 2. Architecture du laboratoire

| Composant | Configuration |
|---|---|
| Machine cliente | Kali Linux |
| Adresse IP cliente | `192.168.195.128` |
| Serveur WAF | Ubuntu Server 22.04.5 LTS |
| Adresse IP du serveur | `192.168.195.146` |
| Reverse proxy | NGINX `1.18.0-6ubuntu14.21` |
| WAF | NAXSI 1.7 |
| Serveur applicatif | Apache HTTP Server |
| Port d'écoute Apache | `127.0.0.1:3001` |
| Application vulnérable | DVWA |
| Base de données | MariaDB |
| Port d'accès client | TCP/80 |

### Schéma logique

```text
             Client Kali Linux
             192.168.195.128
                     |
                     | HTTP - Port 80
                     v
        +----------------------------+
        | Ubuntu Server              |
        | 192.168.195.146             |
        |                            |
        | NGINX + NAXSI               |
        | Reverse proxy / WAF         |
        +----------------------------+
                     |
                     | Proxy local
                     | 127.0.0.1:3001
                     v
        +----------------------------+
        | Apache HTTP Server         |
        | Application DVWA           |
        +----------------------------+
                     |
                     v
                MariaDB
```

Les clients accèdent à DVWA via NGINX. Apache écoute sur l'interface locale afin que les machines du réseau ne puissent pas accéder directement à ce backend par son port applicatif.

## 3. Installation des composants

Les paquets nécessaires ont été installés sur Ubuntu :

```bash
sudo apt update
sudo apt install -y apache2 mariadb-server mariadb-client \
  php libapache2-mod-php php-mysqli php-gd php-curl \
  php-mbstring php-xml git
```

Le code source de DVWA a été récupéré depuis son dépôt officiel :

```bash
cd /tmp
sudo git clone https://github.com/digininja/DVWA.git
sudo cp -a /tmp/DVWA /var/www/html/dvwa
sudo chown -R www-data:www-data /var/www/html/dvwa
```

Les extensions PHP nécessaires au fonctionnement de l'application ont été vérifiées.

## 4. Configuration d'Apache

NGINX utilise le port 80. Apache a donc été configuré pour écouter uniquement sur l'interface locale, sur le port 3001.

Dans `/etc/apache2/ports.conf` :

```apache
Listen 127.0.0.1:3001
```

Dans `/etc/apache2/sites-available/000-default.conf`, le VirtualHost utilise :

```apache
<VirtualHost 127.0.0.1:3001>
    DocumentRoot /var/www/html/dvwa
</VirtualHost>
```

Après modification, la configuration a été vérifiée :

```bash
sudo apache2ctl configtest
sudo systemctl restart apache2
sudo systemctl status apache2
```

Le résultat attendu du contrôle de syntaxe est :

```text
Syntax OK
```

Un avertissement concernant le nom de domaine complet du serveur peut apparaître sans empêcher le démarrage d'Apache.

## 5. Configuration de la base de données

Une base de données dédiée à DVWA a été créée dans MariaDB. Un utilisateur spécifique a été créé et autorisé à accéder à cette base.

Exemple de commandes SQL exécutées avec un compte administrateur MariaDB :

```sql
CREATE DATABASE dvwa;
CREATE USER 'dvwa'@'localhost' IDENTIFIED BY '<MOT_DE_PASSE>';
GRANT ALL PRIVILEGES ON dvwa.* TO 'dvwa'@'localhost';
FLUSH PRIVILEGES;
```

Le mot de passe réel est configuré localement dans `config/config.inc.php`, dans le répertoire de DVWA.

Les paramètres utilisés sont :

```php
$_DVWA[ 'db_server' ]   = '127.0.0.1';
$_DVWA[ 'db_database' ] = 'dvwa';
$_DVWA[ 'db_user' ]     = 'dvwa';
$_DVWA[ 'db_password' ] = '<MOT_DE_PASSE>';
```

Le mot de passe doit rester privé et ne doit pas être publié sur GitHub.

Après configuration, l'initialisation de la base a été effectuée depuis l'interface DVWA avec l'action **Create / Reset Database**.

## 6. Intégration avec NGINX et NAXSI

Le VirtualHost NGINX transmet les requêtes à Apache sur le port local 3001.

Extrait de la configuration utilisée :

```nginx
server {
    listen 80;
    server_name _;

    location / {
        SecRulesEnabled;
        DeniedUrl "/RequestDenied";

        CheckRule "$SQL >= 8" BLOCK;
        CheckRule "$XSS >= 8" BLOCK;
        CheckRule "$RFI >= 8" BLOCK;
        CheckRule "$TRAVERSAL >= 5" BLOCK;
        CheckRule "$EVADE >= 4" BLOCK;

        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location = /RequestDenied {
        internal;
        return 403;
    }
}
```

La configuration a été vérifiée avec :

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Le contrôle de syntaxe doit se terminer par `syntax is ok` et `test is successful`.

## 7. Vérification de l'accès à DVWA

Depuis Kali Linux, l'accès à la page de connexion a été testé via NGINX :

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' \
  http://192.168.195.146/login.php
```

Résultat observé :

```text
HTTP 200
```

Ce résultat confirme que la page de connexion est accessible par l'intermédiaire de NGINX.

L'accès direct au port applicatif depuis Kali a ensuite été testé :

```bash
curl --connect-timeout 3 -s -o /dev/null \
  -w 'HTTP %{http_code}\n' http://192.168.195.146:3001/
```

Résultat observé :

```text
HTTP 000
```

`HTTP 000` signifie que curl n'a reçu aucun code de réponse HTTP. Dans ce test, la connexion directe a échoué, ce qui est cohérent avec l'écoute Apache sur `127.0.0.1:3001`.

## 8. Validation du blocage par NAXSI

Les événements de sécurité ont été consultés dans les journaux NGINX :

```bash
sudo grep 'NAXSI_FMT' /var/log/nginx/error.log | tail -n 20
```

Plusieurs événements comportent `config=block`, indiquant que NAXSI a rejeté les requêtes correspondant aux règles de blocage configurées.

### 8.1 Injection SQL (SQLi)

Requête de test :

```http
GET /?id=1 UNION SELECT 1
```

Extrait du journal :

```text
config=block
cscore0=$SQL
score0=8
zone0=ARGS
id0=1000
var_name0=id
```

La règle `1000` a détecté des éléments correspondant à une injection SQL dans le paramètre `id`. Le score SQL atteint 8, soit le seuil configuré pour le blocage.

### 8.2 Cross-Site Scripting (XSS)

Requête de test :

```http
GET /?q=<script>alert(1)</script>
```

Extrait du journal :

```text
config=block
cscore1=$XSS
score1=8
zone0=ARGS
id0=1010
var_name0=q
```

La règle `1010` a déclenché un score XSS de 8 dans le paramètre `q`. Le seuil de blocage est atteint.

### 8.3 Remote File Inclusion (RFI)

Requête de test :

```http
GET /?url=http://example.com
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

La règle `1100` a détecté la chaîne correspondant au motif RFI dans le paramètre `url`.

### 8.4 Directory Traversal

Requête de test :

```http
GET /?file=../../etc/passwd
```

Extrait du journal :

```text
config=block
cscore0=$TRAVERSAL
score0=8
zone0=ARGS
id0=1200
var_name0=file
```

La règle `1200` a détecté une séquence associée à la traversée de répertoires dans le paramètre `file`.

### Synthèse des résultats

| Catégorie | Identifiant de règle | Score observé | État |
|---|---:|---:|---|
| SQL Injection | 1000 | `$SQL = 8` | Bloqué |
| XSS | 1010 | `$XSS = 8` | Bloqué |
| RFI | 1100 | `$RFI = 8` | Bloqué |
| Directory Traversal | 1200 | `$TRAVERSAL = 8` | Bloqué |

Ces événements démontrent le déclenchement des règles NAXSI et le blocage de ces requêtes de test. Ils ne constituent pas, à eux seuls, une évaluation exhaustive de la protection contre toutes les variantes de ces attaques.

## 9. Conclusion

Cette phase a permis de déployer DVWA sur Ubuntu, de configurer Apache et MariaDB, puis de placer l'application derrière NGINX et NAXSI.

Les tests confirment l'accessibilité de DVWA via le port 80, l'échec de l'accès direct au port applicatif depuis Kali et la présence d'événements de blocage NAXSI pour des requêtes SQLi, XSS, RFI et Directory Traversal.

La phase suivante portera sur une campagne de tests de sécurité Web plus structurée, avec comparaison des réponses HTTP, analyse des journaux, identification des faux positifs et documentation des limites du WAF.
