# Install KiwiIRC and UnrealIRCd on Ubuntu 24.04 LTS

This guide shows a minimal, practical setup: install KiwiIRC (web client/gateway), UnrealIRCd (IRC server), then secure KiwiIRC with HTTPS via Nginx + Let's Encrypt (standalone). Adjust usernames, hostnames and paths to your environment.

## 0. Prerequisites
- Ubuntu 24.04 LTS
- A domain name pointing to the server (for TLS)
- Ports 80 and 443 reachable (for certbot standalone)
- Basic Linux admin privileges (sudo)

## 1. System update and build deps
```sh
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential pkg-config gdb libssl-dev libpcre2-dev \
  libargon2-dev libsodium-dev libc-ares-dev libcurl4-openssl-dev \
  curl gnupg2 ca-certificates lsb-release ubuntu-keyring wget certbot
```

## 2. Install KiwiIRC (prebuilt .deb)
Download and install the chosen release (example v1.7.1):
```sh
wget https://github.com/kiwiirc/kiwiirc/releases/download/v1.7.1/kiwiirc-server_v1.7.1-2_linux_amd64.deb
sudo dpkg -i kiwiirc-server_v1.7.1-2_linux_amd64.deb
# if dpkg reports missing deps:
sudo apt-get -f install -y
```

Create or edit the systemd unit if the package didn't provide a working one:
```ini
// filepath: /etc/systemd/system/kiwiirc.service
[Unit]
Description=KiwiIRC
After=network.target

[Service]
ExecStart=/usr/bin/kiwiirc --config /etc/kiwiirc/config.conf
WorkingDirectory=/etc/kiwiirc
Restart=always
User=www-data
Group=www-data
Environment=PATH=/usr/bin:/usr/local/bin
Environment=NODE_ENV=production

[Install]
WantedBy=multi-user.target
```
Reload systemd and do not start yet:
```sh
sudo systemctl daemon-reload
sudo systemctl disable --now kiwiirc || true
```

## 3. Configure KiwiIRC
Edit the main config:
```toml
// filepath: /etc/kiwiirc/config.conf
# The name of this gateway as reported in WEBIRC to IRC servers
gateway_name = "webircgateway"

# A secret string used for generating client JWT tokens. Do not share this!
secret = "<generate-with-openssl>"

# The websocket / http server (use unprivileged port)
[server.1]
bind = "0.0.0.0"
port = 8080
```
Generate a secret:
```sh
openssl rand -base64 32
# paste the output into `secret = "..."` in config.conf
```

Start KiwiIRC:
```sh
sudo systemctl enable --now kiwiirc
sudo systemctl status kiwiirc
```
If it fails to start, check journalctl for errors:
```sh
sudo journalctl -u kiwiirc -b --no-pager
```

Open http://SERVER_IP:8080 to verify the web UI loads (it may not connect to IRC yet).

## 4. Install and build UnrealIRCd
Download, extract, configure and install (example 6.2.1):
```sh
wget https://www.unrealircd.org/downloads/unrealircd-6.2.1.tar.gz
tar -xvzf unrealircd-6.2.1.tar.gz
cd unrealircd-6.2.1
./Config    # accept defaults; enable OpenSSL (self-signed ok for internal)
make
sudo make install
```

Create a password hash for OPER and other passwords:
```sh
./unrealircd mkpasswd
# copy the resulting hash (argon2 or other) for use in unrealircd.conf
```

Edit UnrealIRCd config:
```conf
// filepath: /home/youruser/unrealircd-6.2.1/conf/unrealircd.conf
# ...existing code...
oper admin {
    mask *;
    password "$argon2$5$abce34fg....";  # paste the mkpasswd output
    class clients;
}
# ...existing code...

# Add 3 cloak keys (80+ char base64 strings)
cloak-keys {
    "BASE64-KEY-1...";
    "BASE64-KEY-2...";
    "BASE64-KEY-3...";
}
# ...existing code...
```
Generate 80-character base64 keys:
```sh
openssl rand -base64 80
```

Start UnrealIRCd and verify it runs:
```sh
cd /home/youruser/unrealircd-6.2.1
./unrealircd start
# view logs in the unrealircd logs folder for startup info/errors
```

## 5. Connect KiwiIRC to UnrealIRCd (backend)
Edit KiwiIRC upstream settings to point to your IRC server (localhost or internal IP):
```toml
// filepath: /etc/kiwiirc/config.conf
[upstream.1]
hostname = "localhost"
port = 6667
tls = false
```
Restart KiwiIRC:
```sh
sudo systemctl restart kiwiirc
```
Now the web UI should be able to connect to the IRC server.

## 6. Secure KiwiIRC with Nginx reverse proxy + Let's Encrypt
Install Nginx from upstream repo (optional, recommended steps included):
```sh
# Import key
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
  | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null

# Add repo (stable)
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" \
  | sudo tee /etc/apt/sources.list.d/nginx.list

# Pin packages to prefer nginx.org
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
  | sudo tee /etc/apt/preferences.d/99nginx

sudo apt update
sudo apt install -y nginx
```

Do not leave nginx running while using certbot standalone. Stop Nginx, then request certs:
```sh
sudo systemctl stop nginx
sudo certbot certonly --standalone -d irc.example.com
# certificates placed at /etc/letsencrypt/live/irc.example.com/
```

Create an Nginx site for KiwiIRC:
```nginx
// filepath: /etc/nginx/conf.d/kiwiirc.conf
upstream kiwirc {
    server 127.0.0.1:8080;
}

server {
    listen 80;
    server_name irc.example.com;

    # Redirect HTTP to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    server_name irc.example.com;

    ssl_certificate /etc/letsencrypt/live/irc.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/irc.example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://kiwirc;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```
Start nginx:
```sh
sudo systemctl start nginx
sudo systemctl enable nginx
```

Visit https://irc.example.com — the page should load over HTTPS and the KiwiIRC frontend will connect to UnrealIRCd.

## 7. Notes & troubleshooting
- If dpkg installation fails, run `sudo apt-get -f install -y`.
- After editing systemd unit files: `sudo systemctl daemon-reload`.
- KiwiIRC runs as the configured user (e.g., www-data); ensure /etc/kiwiirc and config files are readable by that user.
- If certbot fails because ports are blocked, open ports 80 and 443 or use DNS challenge.
- Monitor logs:
  - KiwiIRC: `sudo journalctl -u kiwiirc -f`
  - Nginx: `/var/log/nginx/error.log` and `access.log`
  - UnrealIRCd: logs in its install directory

This is a concise, reproducible flow. Adjust values (usernames, domain, paths) for your deployment.
