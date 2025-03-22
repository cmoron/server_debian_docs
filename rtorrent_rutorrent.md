# ruTorrent + rTorrent installation with SCGI socket on Debian 12

## 🧱 Stack Overview

- **OS**: Debian 12 (Bookworm)
- **rTorrent**: Installed via APT, run by user `cyril`
- **ruTorrent**: Cloned in `/home/cyril/src/ruTorrent`, served by Nginx
- **Nginx**: Acts as web server and SCGI reverse proxy
- **PHP-FPM**: Handles ruTorrent's PHP scripts (v8.2)
- **Communication**: SCGI via local UNIX socket

---

## ✅ rTorrent Configuration

**File:** `/home/cyril/.rtorrent.rc`

```ini
# Session path (stores torrent state)
session = /home/cyril/rtorrent/.session

# Enable local UNIX socket for SCGI
network.scgi.open_local = (cat,(session.path),rpc.socket)
execute.nothrow = chmod,770,(cat,(session.path),rpc.socket)

# Optional fixed port (useful for firewall/NAT)
port_range = 49160-49160
port_random = no
```

🔁 rTorrent is launched **manually** (no systemd):

```bash
rtorrent
```

---

## ✅ UNIX Permissions

To allow ruTorrent (PHP as `www-data`) to access the socket:

```bash
sudo usermod -a -G cyril www-data
sudo chmod 770 /home/cyril/rtorrent/.session
sudo systemctl restart php8.2-fpm
```

---

## ✅ ruTorrent Configuration

**File:** `/var/www/rutorrent/conf/config.php`

```php
$scgi_port = 0;
$scgi_host = "unix:///home/cyril/rtorrent/.session/rpc.socket";
```

🔁 Plugins causing warnings can be disabled via `plugins.ini`

---

## ✅ Nginx Configuration

**File:** `/etc/nginx/sites-available/rutorrent.moron.at`

```nginx
server {
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
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/rutorrent.moron.at/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/rutorrent.moron.at/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}
```

---

## ✅ Testing

- Visit: `https://rutorrent.moron.at`
- Check if rTorrent is connected (green icon)
- Add test torrents (e.g. Debian ISO)

---

## 📌 Notes

- Socket path **must match** between rTorrent and ruTorrent
- Restart `php-fpm` if user group is modified
- No `system.daemon.set = true` used for simplicity
- rTorrent is started manually inside a `screen` session

---

## 🧪 Optional: Manual RPC test

Use `socat` to send raw XML-RPC request via socket (see chat logs for details)

---

## 🔒 Security

- SCGI is local-only (not exposed on TCP)
- SSL enabled via Certbot / Let's Encrypt
- Access to socket restricted by UNIX permissions

---

## ✅ Done!

A clean, modern, secure ruTorrent + rTorrent setup using UNIX sockets and nginx reverse proxy.

