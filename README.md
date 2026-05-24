<h1 align="center">🔐 Oxidized + Nginx + Keycloak SSO (oauth2-proxy)</h1>

<p align="center">
  Déploiement d'Oxidized en production avec reverse proxy Nginx et authentification SSO Keycloak via oauth2-proxy, fédérée sur un Active Directory.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Oxidized-systemd-blue" />
  <img src="https://img.shields.io/badge/Keycloak-Docker-orange" />
  <img src="https://img.shields.io/badge/Auth-OAuth2%20%2F%20OIDC-green" />
  <img src="https://img.shields.io/badge/OS-Debian-red" />
  <img src="https://img.shields.io/badge/Langue-Français-lightgrey" />
  <img src="https://img.shields.io/badge/Licence-MIT-lightgrey" />
</p>

---

## Contexte

Oxidized sauvegarde automatiquement les configurations des équipements réseau (Cisco IOS, Cisco SMB, Huawei VRP) toutes les 12 heures via SSH/Telnet, et versionne les configs dans un dépôt Git local. L'interface web est exposée en HTTPS et protégée par une authentification SSO Keycloak, fédérée sur l'Active Directory de l'entreprise.

**Stack utilisée :** Oxidized · Nginx · Keycloak · oauth2-proxy · Docker · Active Directory / LDAP · Debian

---

## Architecture

```
Utilisateur
    │
    │ HTTPS (443)
    ▼
 Nginx (reverse proxy)
    │
    ├──► oauth2-proxy (127.0.0.1:4180)
    │         │
    │         └──► Keycloak (127.0.0.1:8080) ──► Active Directory (LDAP)
    │
    └──► Oxidized-web (127.0.0.1:8888)
```

**Flux d'authentification :**
1. L'utilisateur accède à `https://oxidized.example.com`
2. Nginx interroge oauth2-proxy pour vérifier la session
3. Si pas de session → redirection vers Keycloak (`https://keycloak.example.com`)
4. Keycloak authentifie l'utilisateur via LDAP (Active Directory)
5. Après auth réussie → création d'un cookie de session sécurisé
6. Nginx transmet la requête à Oxidized (`127.0.0.1:8888`)

---

## Prérequis

### DNS
Créer deux enregistrements dans votre DNS (AD DNS) :
```
oxidized.example.com  → IP du serveur
keycloak.example.com  → IP du serveur
```

### Certificats (self-signed accepté)

```bash
sudo mkdir -p /etc/ssl/certs/custom

# Certificat Keycloak
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 825 \
  -keyout /etc/ssl/certs/custom/keycloak.key \
  -out    /etc/ssl/certs/custom/keycloak.crt \
  -subj   "/CN=keycloak.example.com" \
  -addext "subjectAltName=DNS:keycloak.example.com"

# Certificat Oxidized
sudo openssl req -x509 -nodes -newkey rsa:2048 -days 825 \
  -keyout /etc/ssl/certs/custom/oxidized.key \
  -out    /etc/ssl/certs/custom/oxidized.crt \
  -subj   "/CN=oxidized.example.com" \
  -addext "subjectAltName=DNS:oxidized.example.com"

sudo chmod 600 /etc/ssl/certs/custom/*.key
```

### Docker

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Utilisateur AD dédié
Créer un compte `keycloak_ldap` dans l'Active Directory (mot de passe sans expiration). Il sera utilisé par Keycloak pour les requêtes LDAP en lecture seule.

---

## 1. Installation d'Oxidized

### 1.1 Dépendances

```bash
sudo apt update
sudo apt install -y ruby ruby-dev libsqlite3-dev libssl-dev pkg-config cmake \
  libssh2-1-dev libicu-dev zlib1g-dev g++ libyaml-dev libzstd-dev git
```

### 1.2 Installation des gems

```bash
sudo gem install oxidized
sudo gem install oxidized-web
```

### 1.3 Utilisateur dédié et dossiers

```bash
sudo useradd -s /bin/bash -m oxidized
sudo mkdir -p /etc/oxidized
sudo chown -R oxidized:oxidized /etc/oxidized
sudo -u oxidized mkdir -p /etc/oxidized/{crashes,log,configs}
```

### 1.4 Fichier d'inventaire — `router.db`

Chaque ligne correspond à un équipement : `IP:modèle` (ou `IP:modèle:telnet` pour forcer Telnet).

```bash
sudo -u oxidized nano /etc/oxidized/router.db
```

```
# Exemple d'inventaire
192.168.1.1:ios
192.168.1.2:ios
192.168.1.3:ciscosmb
192.168.1.4:ios:telnet
192.168.1.5:vrp
```

| Modèle     | Équipement                        |
|------------|-----------------------------------|
| `ios`      | Cisco Catalyst (IOS)              |
| `ciscosmb` | Cisco Small Business              |
| `vrp`      | Huawei                            |

> Pour ajouter un switch : ajouter une ligne et cliquer sur **"Reload node list from sourced backend"** dans l'interface web.

### 1.5 Configuration Oxidized — `config`

```bash
sudo -u oxidized nano /etc/oxidized/config
```

```yaml
---
username: "votre_utilisateur_reseau"
password: "votre_mot_de_passe"
model: "ios"
resolve_dns: true
interval: 43200        # Sauvegarde toutes les 12h
use_max_threads: false
threads: 30
timeout: 20
timelimit: 300
retries: 3
prompt: !ruby/regexp /^([\w.@-]+[#>]\s?)$/

crash:
  directory: "/etc/oxidized/crashes"
  hostnames: false

extensions:
  oxidized-web:
    load: true
    listen: 127.0.0.1
    port: 8888

input:
  default: ssh
  ssh:
    secure: false
  telnet: {}

output:
  default: git
  git:
    user: "Oxidized"
    email: "oxidized@local"
    repo: "/etc/oxidized/repository.git"

source:
  default: csv
  csv:
    file: "/etc/oxidized/router.db"
    delimiter: !ruby/regexp /:/
    map:
      name: 0
      model: 1
      input: 2
    model_map:
      cisco: ios
      juniper: junos
      huawei: vrp
```

### 1.6 Initialiser le dépôt Git

```bash
sudo -u oxidized bash -lc 'cd /etc/oxidized && git init --bare repository.git'
```

### 1.7 Test manuel

```bash
sudo -u oxidized env OXIDIZED_HOME=/etc/oxidized oxidized
# Ctrl+C pour arrêter
```

### 1.8 Service systemd

```bash
sudo nano /etc/systemd/system/oxidized.service
```

```ini
[Unit]
Description=Oxidized - Network Device Configuration Backup
After=network-online.target
Wants=network-online.target

[Service]
User=oxidized
Group=oxidized
Environment=OXIDIZED_HOME=/etc/oxidized
ExecStart=/usr/local/bin/oxidized
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now oxidized
sudo systemctl status oxidized --no-pager
```

---

## 2. Installation Nginx

```bash
sudo apt install -y nginx
nginx -t
systemctl status nginx --no-pager
```

### Vhost Keycloak — `/etc/nginx/sites-available/keycloak`

```nginx
server {
    listen 443 ssl;
    server_name keycloak.example.com;

    ssl_certificate     /etc/ssl/certs/custom/keycloak.crt;
    ssl_certificate_key /etc/ssl/certs/custom/keycloak.key;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  443;
    }
}

server {
    listen 80;
    server_name keycloak.example.com;
    return 301 https://$host$request_uri;
}
```

### Vhost Oxidized — `/etc/nginx/sites-available/oxidized`

```nginx
server {
    listen 443 ssl;
    server_name oxidized.example.com;

    ssl_certificate     /etc/ssl/certs/custom/oxidized.crt;
    ssl_certificate_key /etc/ssl/certs/custom/oxidized.key;

    # Endpoints oauth2-proxy
    location /oauth2/ {
        proxy_pass http://127.0.0.1:4180;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
    }

    location = /oauth2/auth {
        proxy_pass http://127.0.0.1:4180/oauth2/auth;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header Content-Length    "";
        proxy_pass_request_body off;
    }

    # Application protégée
    location / {
        auth_request /oauth2/auth;
        error_page 401 = /oauth2/sign_in;

        proxy_pass http://127.0.0.1:8888;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}

server {
    listen 80;
    server_name oxidized.example.com;
    return 301 https://$host$request_uri;
}
```

```bash
ln -s /etc/nginx/sites-available/keycloak /etc/nginx/sites-enabled/keycloak
ln -s /etc/nginx/sites-available/oxidized /etc/nginx/sites-enabled/oxidized
nginx -t && systemctl reload nginx
```

---

## 3. Déploiement Keycloak (Docker)

### `/opt/keycloak/docker-compose.yml`

```yaml
services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      POSTGRES_DB:       keycloak
      POSTGRES_USER:     keycloak
      POSTGRES_PASSWORD: "<POSTGRES_PASSWORD>"
    volumes:
      - ./postgres_data:/var/lib/postgresql/data

  keycloak:
    image: quay.io/keycloak/keycloak:latest
    restart: unless-stopped
    environment:
      KC_DB:                   postgres
      KC_DB_URL:               jdbc:postgresql://postgres:5432/keycloak
      KC_DB_USERNAME:          keycloak
      KC_DB_PASSWORD:          "<POSTGRES_PASSWORD>"
      KC_HTTP_ENABLED:         "true"
      KC_PROXY:                edge
      KC_HOSTNAME:             keycloak.example.com
      KC_HOSTNAME_STRICT_HTTPS: "true"
      KC_HOSTNAME_STRICT:      "true"
      KEYCLOAK_ADMIN:          "<KEYCLOAK_ADMIN_USER>"
      KEYCLOAK_ADMIN_PASSWORD: "<KEYCLOAK_ADMIN_PASSWORD>"
    command: ["start", "--optimized"]
    depends_on:
      - postgres
    ports:
      - "127.0.0.1:8080:8080"
```

```bash
cd /opt/keycloak && docker compose up -d
# Vérification endpoint OIDC
curl -kI https://keycloak.example.com/realms/MON_REALM/.well-known/openid-configuration
```

### Configuration du Realm

1. Se connecter à `https://keycloak.example.com` avec les credentials admin
2. **Manage realms → Create realm** → nommer le realm (ex : `MON_REALM`)

**Fédération LDAP (Active Directory) :**
- User federation → Add provider → `ldap`
- Vendor : `Active Directory`
- Connection URL : `ldap://IP_CONTROLEUR_DE_DOMAINE`
- Bind DN : `CN=keycloak_ldap,CN=Users,DC=domaine,DC=local`
- Users DN : `OU=<votre_OU>,DC=domaine,DC=local`
- Username LDAP attribute : `sAMAccountName`
- Edit mode : `READ_ONLY`
- Après save → **Action → Sync all users** pour vérifier la synchronisation

**Client OIDC pour oauth2-proxy :**
- Clients → Create client
- Client ID : `oxidized`
- Type : `OpenID Connect` / Confidential client : ON
- Valid redirect URIs : `https://oxidized.example.com/oauth2/callback`
- Web origins : `https://oxidized.example.com`
- Récupérer le **Client Secret** dans l'onglet `Credentials`

---

## 4. Déploiement oauth2-proxy (Docker)

### Générer le cookie secret

```bash
python3 -c "import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode().rstrip('='))"
```

### `/opt/oauth2-proxy/docker-compose.yml`

```yaml
services:
  oauth2-proxy:
    image: quay.io/oauth2-proxy/oauth2-proxy:v7.14.1
    restart: unless-stopped
    network_mode: "host"
    command:
      - --provider=keycloak-oidc
      - --http-address=127.0.0.1:4180
      - --oidc-issuer-url=https://keycloak.example.com/realms/MON_REALM
      - --client-id=oxidized
      - --client-secret=<CLIENT_SECRET_KEYCLOAK>
      - --redirect-url=https://oxidized.example.com/oauth2/callback
      - --email-domain=*
      - --ssl-insecure-skip-verify=true
      - --oidc-email-claim=preferred_username
      - --insecure-oidc-allow-unverified-email=true
      - --cookie-secret=<COOKIE_SECRET_BASE64>
      - --cookie-secure=true
      - --cookie-samesite=lax
      - --cookie-path=/
      - --reverse-proxy=true
      - --set-xauthrequest=true
```

```bash
cd /opt/oauth2-proxy && docker compose up -d

# Vérifications
curl -I http://127.0.0.1:4180/ping   # Attendu : 200 OK
ss -lntp | grep 4180                 # Attendu : 127.0.0.1:4180
```

### Test final

```bash
# Ouvrir dans un navigateur :
https://oxidized.example.com
# → Doit rediriger vers la page de login Keycloak
```

---

## Maintenance

### Oxidized (systemd)

```bash
sudo systemctl status oxidized --no-pager
sudo systemctl restart oxidized
sudo journalctl -u oxidized -f
```

### Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### oauth2-proxy (Docker)

```bash
cd /opt/oauth2-proxy
docker compose ps
docker compose logs -f --tail=200 oauth2-proxy
docker compose restart oauth2-proxy
```

### Keycloak (Docker)

```bash
cd /opt/keycloak
docker compose ps
docker compose logs -f --tail=200 keycloak
docker compose restart keycloak
```

### Diagnostic rapide — ports attendus

```bash
sudo ss -lntp | egrep '(:8888|:4180|:8080|:443|:80)'
```

| Service      | Port attendu        |
|--------------|---------------------|
| Oxidized     | `127.0.0.1:8888`    |
| oauth2-proxy | `127.0.0.1:4180`    |
| Keycloak     | `127.0.0.1:8080`    |
| Nginx        | `0.0.0.0:80` et `0.0.0.0:443` |

### Mises à jour

```bash
# oauth2-proxy
cd /opt/oauth2-proxy && docker compose pull && docker compose up -d

# Keycloak
cd /opt/keycloak && docker compose pull && docker compose up -d

# Oxidized
sudo systemctl stop oxidized
sudo gem update oxidized oxidized-web
sudo systemctl start oxidized
```

---

## Emplacements importants

| Élément                  | Chemin                                          |
|--------------------------|-------------------------------------------------|
| Config Oxidized          | `/etc/oxidized/config`                          |
| Inventaire équipements   | `/etc/oxidized/router.db`                       |
| Dépôt Git des configs    | `/etc/oxidized/repository.git`                  |
| Logs Oxidized            | `sudo journalctl -u oxidized`                   |
| Vhosts Nginx             | `/etc/nginx/sites-available/{oxidized,keycloak}`|
| Compose oauth2-proxy     | `/opt/oauth2-proxy/docker-compose.yml`          |
| Compose Keycloak         | `/opt/keycloak/docker-compose.yml`              |
| Certificats              | `/etc/ssl/certs/custom/`                        |
