# 🛠 README - Déploiement d'un serveur Debian 12 (moron.at)

Ce document récapitule toutes les étapes de configuration et de déploiement d'un serveur Debian 12 hébergeant les services suivants :

- Page d'accueil statique (homepage)
- Site mypacer.fr (FastAPI + Svelte + PostgreSQL)
- Interface ruTorrent avec rTorrent
- n8n (automatisation low-code)
- Nginx avec certificats Let's Encrypt

---

## 🧱 Prérequis

- Serveur Debian 12 installé
- Accès root ou sudo
- Accès DNS configuré pour :
  - moron.at / [www.moron.at](http://www.moron.at)
  - api.mypacer.fr / mypacer.fr
  - rutorrent.moron.at
  - n8n.moron.at

---

## 👤 Création de l'utilisateur principal

```bash
adduser cyril
usermod -aG sudo cyril
usermod -aG docker cyril  # si Docker installé
```

Copier les clés SSH dans `/home/cyril/.ssh/authorized_keys` si nécessaire.

---

## 🧰 Paquets à installer

```bash
apt update && apt upgrade -y
apt install -y \
  sudo git git-lfs curl unzip screen nginx php-fpm php-cli php-curl php-xml php-mbstring php-json \
  php8.2-fpm python3-certbot-nginx certbot bind9 docker.io docker-compose \
  fzf ripgrep
```

## 📥 Initialiser Git LFS

Avant de cloner certains dépôts (comme `running_pace_api`), pensez à initialiser Git LFS :

```bash
git lfs install
```

Activer les services :

```bash
systemctl enable --now nginx php8.2-fpm docker
```

---

## 🔐 Clé SSH pour GitLab

En tant qu'utilisateur `cyril` :

```bash
ssh-keygen -t rsa -b 4096 -C "cyril@gitlab"
cat ~/.ssh/id_rsa.pub
```

Ajouter la clé sur GitLab via **Settings > SSH Keys**.

Tester :

```bash
ssh -T git@gitlab.com
```

---

## 🧾 Configuration personnelle

Un répertoire `~/etc` est utilisé pour stocker la configuration personnelle. Les dépôts sont clonés via SSH :

```bash
mkdir -p ~/etc && cd ~/etc
git clone --branch SERVER git@github.com:cmoron/dotfiles.git
git clone git@github.com:cmoron/vimcfg.git
```

Créer les liens symboliques :

```bash
ln -sf ~/etc/vimcfg/.vim/ ~/.vim
ln -sf ~/etc/dotfiles/.bashrc ~/.bashrc
ln -sf ~/etc/dotfiles/.config/git/ ~/.config/git
mkdir -p ~/shell && ln -sf ~/etc/dotfiles/.local/shell/fzf/ ~/shell/fzf
```

---

## 📁 Structure des projets

Tous les dépôts sont clonés dans : `/home/cyril/src/`

```bash
mkdir -p /home/cyril/src && cd /home/cyril/src
```

Cloner les dépôts :

```bash
git clone git@github.com:cmoron/homepage.git
git clone git@github.com:cmoron/running_pace_api.git
git clone git@github.com:cmoron/running_pace_table.git
git clone https://github.com/Novik/ruTorrent.git
mkdir n8n  # déployé manuellement en Docker
```

---

## 🌐 Nginx & Certbot

### 📂 Préparation du répertoire ACME

```bash
sudo mkdir -p /var/www/letsencrypt
sudo chown -R www-data:www-data /var/www/letsencrypt
```

### 📂 Gestion des droits nginx sur les répertoires projets

```
sudo usermod -aG cyril www-data
sudo chmod 750 /home/cyril/
```

### ⚙️ Configuration initiale des virtual hosts (HTTP)

Avant de lancer Certbot, il faut créer un fichier Nginx par domaine dans `/etc/nginx/sites-available/` avec un minimum de configuration permettant la validation HTTP.

Voici les blocs à insérer dans chaque fichier :

#### `/etc/nginx/sites-available/moron.at`

```nginx
server {
    listen 80;
    server_name moron.at www.moron.at;
    root /var/www/letsencrypt;

    location /.well-known/acme-challenge/ {
        allow all;
    }
}
```

#### `/etc/nginx/sites-available/mypacer.fr`

```nginx
server {
    listen 80;
    server_name mypacer.fr www.mypacer.fr;
    root /var/www/letsencrypt;

    location /.well-known/acme-challenge/ {
        allow all;
    }
}
```

#### `/etc/nginx/sites-available/api.mypacer.fr`

```nginx
server {
    listen 80;
    server_name api.mypacer.fr;
    root /var/www/letsencrypt;

    location /.well-known/acme-challenge/ {
        allow all;
    }
}
```

#### `/etc/nginx/sites-available/rutorrent.moron.at`

```nginx
server {
    listen 80;
    server_name rutorrent.moron.at;
    root /var/www/letsencrypt;

    location /.well-known/acme-challenge/ {
        allow all;
    }
}
```

#### `/etc/nginx/sites-available/n8n.moron.at`

```nginx
server {
    listen 80;
    server_name n8n.moron.at;
    root /var/www/letsencrypt;

    location /.well-known/acme-challenge/ {
        allow all;
    }
}
```

### 🔗 Activer les sites

```bash
cd /etc/nginx/sites-enabled/
sudo ln -s ../sites-available/moron.at
sudo ln -s ../sites-available/mypacer.fr
sudo ln -s ../sites-available/api.mypacer.fr
sudo ln -s ../sites-available/rutorrent.moron.at
sudo ln -s ../sites-available/n8n.moron.at
```

### 🚀 Redémarrer Nginx

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 🔐 Génération des certificats Let's Encrypt

Utiliser Certbot en fonction du plugin supporté :

```bash
# Homepage
sudo certbot --nginx -d moron.at -d www.moron.at

# mypacer front
sudo certbot --nginx -d mypacer.fr -d www.mypacer.fr

# mypacer API (webroot car le proxy vers uvicorn empêche le test HTTP)
sudo certbot certonly --webroot -w /var/www/letsencrypt -d api.mypacer.fr

# ruTorrent
sudo certbot --nginx -d rutorrent.moron.at

# n8n
sudo certbot certonly --webroot -w /var/www/letsencrypt -d n8n.moron.at
```

Une fois tous les certificats générés, remplacer la configuration HTTP par une configuration HTTPS propre avec reverse proxy ou fichiers statiques selon les cas.

---

## 🌐 Page d'accueil (moron.at)

- Projet HTML statique dans `/home/cyril/src/homepage`
- Servi directement par nginx avec root pointant sur ce répertoire
- Accessible en HTTPS via moron.at et [www.moron.at](http://www.moron.at)

```nginx
# /etc/nginx/sites-available/moron.at
server {
    listen 80;
    listen [::]:80;
    server_name moron.at www.moron.at;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name moron.at www.moron.at;

    ssl_certificate /etc/letsencrypt/live/moron.at/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/moron.at/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    root /home/cyril/src/homepage;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

## 🏃‍♂️ mypacer.fr

### Backend FastAPI (api.mypacer.fr)

- Projet : `running_pace_api`
- Dockerisé via `docker-compose`
- Container : `running-pace-api`
- Exposé en interne sur le port `8000`
- Accès externe via nginx (reverse proxy)
- Variables définies via `.env`

```nginx
# /etc/nginx/sites-available/api.mypacer.fr
server {
    listen 80;
    server_name api.mypacer.fr;

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name api.mypacer.fr;

    ssl_certificate /etc/letsencrypt/live/api.mypacer.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.mypacer.fr/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass http://127.0.0.1:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### PostgreSQL

- Container `postgres`
- Volume Docker : `postgres_data`
- DB initialisée via `init.sql` (vide ou avec import manuel)

### Frontend Svelte (mypacer.fr)

- Projet : `running_pace_table`
- Build du projet avec `npm run build`
- Fichiers statiques copiés dans un répertoire nginx (par exemple `/var/www/mypacer_front`)
- Servi via nginx sur mypacer.fr / [www.mypacer.fr](http://www.mypacer.fr)

```nginx
# /etc/nginx/sites-available/mypacer.fr
server {
    server_name mypacer.fr www.mypacer.fr;

    root /home/cyril/src/running_pace_table/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/mypacer.fr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/mypacer.fr/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name mypacer.fr www.mypacer.fr;
    return 301 https://$host$request_uri;
}
```

---

## 🔁 rtorrent + ruTorrent (rutorrent.moron.at)

- rTorrent lancé manuellement dans un `screen`
- Utilise SCGI via socket UNIX : `/home/cyril/rtorrent/.session/rpc.socket`
- ruTorrent cloné dans `/home/cyril/src/ruTorrent`

```nginx
# /etc/nginx/sites-available/rutorrent.moron.at
server {
    listen 443 ssl;
    server_name rutorrent.moron.at;

    root /home/cyril/src/ruTorrent;
    index index.html index.php;

    location / {
        try_files $uri $uri/ =404;
    }

    location /RPC2 {
        include scgi_params;
        scgi_pass unix:/home/cyril/rtorrent/.session/rpc.socket;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }

    ssl_certificate /etc/letsencrypt/live/rutorrent.moron.at/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rutorrent.moron.at/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name rutorrent.moron.at;
    return 301 https://$host$request_uri;
}
```

---

## ⚙️ n8n (n8n.moron.at)

- Déployé via Docker (non dockerisé dans git)
- Conteneur mappé sur port `5678`
- Nginx proxy pass vers `http://localhost:5678`
- Certbot via `certonly` avant config nginx
- Volume Docker : `n8n_data:/home/node/.n8n`

```nginx
# /etc/nginx/sites-available/n8n.moron.at
server {
    listen 443 ssl;
    server_name n8n.moron.at;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    ssl_certificate /etc/letsencrypt/live/n8n.moron.at/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.moron.at/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name n8n.moron.at;
    return 301 https://$host$request_uri;
}
```

---

## 🧪 Vérifications

- Tous les domaines répondent en HTTPS
- `curl https://api.mypacer.fr` retourne une réponse JSON
- `rutorrent.moron.at` affiche les torrents actifs
- `n8n.moron.at` demande une création de compte (si vierge)

---

## 🔐 Protection par mot de passe (ruTorrent)

Installer l'utilitaire :

```bash
sudo apt install apache2-utils
```

Créer le fichier de mot de passe :

```bash
sudo htpasswd -c /etc/nginx/.htpasswd cyril
```

Modifier la configuration Nginx `rutorrent.moron.at` :

```nginx
location / {
    try_files $uri $uri/ =404;
    auth_basic "Restricted Area";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

Recharger Nginx :

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 📦 À faire après installation

- Peupler la base mypacer avec les scripts de scraping
- Ajouter une tâche cron pour la mise à jour automatique
- Sécuriser davantage (pare-feu, backups, logs, etc.)

---

## 📌 Notes

- Serveur sans `systemd` pour rtorrent (screen préféré)
- DNS propagé via Cloudflare ou registrar avec TTL faible
- Scripts personnalisés dans `/home/cyril/dev/` pour backups ou crawlers

---

