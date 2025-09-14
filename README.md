# n8n on Oracle Cloud Free Tier (Ubuntu 24.04)

> **Production‑ready on a shoestring**: This repo deploys **n8n** on an **Oracle Cloud Free Tier** VM using **Docker Compose**, with **NGINX** as a reverse proxy and **Let’s Encrypt (Certbot)** TLS. 

---

## Table of contents

* [Architecture](#architecture)
* [Prerequisites](#prerequisites)
* [Quick start](#quick-start)
* [Detailed steps](#detailed-steps)

  * [1) Create the VM on OCI](#1-create-the-vm-on-oci)
  * [2) Cloudflare DNS](#2-cloudflare-dns)
  * [3) Bootstrap the VM](#3-bootstrap-the-vm)
  * [4) Issue TLS with Certbot](#4-issue-tls-with-certbot)
  * [5) (Optional) Cloudflare proxy + real client IPs](#5-optional-cloudflare-proxy--real-client-ips)
* [Operations](#operations)

  * [Upgrade n8n](#upgrade-n8n)
  * [Backup & restore](#backup--restore)
  * [Troubleshooting](#troubleshooting)
* [Security & what not to commit](#security--what-not-to-commit)
* [Repo layout](#repo-layout)


---

## Architecture

```
Internet ──▶ Cloudflare (orange/proxied) ──▶ NGINX on VM ──▶ n8n (Docker, 127.0.0.1:5678)
                               ▲
                               └── Let’s Encrypt (HTTP‑01 or DNS‑01 via CF)
```

* VM: **Ubuntu 24.04** on **VM.Standard.E2.1.Micro** (Always Free), region `eu-amsterdam-1`.
* NGINX terminates TLS, proxies to n8n bound only to **localhost:5678**.
* Certbot installs & auto‑renews TLS.

---

## Prerequisites

* Oracle Cloud account, Free Tier enabled.
* A domain in **Cloudflare** (e.g. `n8n.example.com`).
* SSH access to the VM as `ubuntu`.

---

## Quick start

> Replace values (domain/email) before running.

```bash
# SSH in
ssh ubuntu@<your-vm-public-ip>

# Save the bootstrap script
cat <<'EOF' > bootstrap_n8n.sh
#!/usr/bin/env bash
# bootstrap_n8n.sh — n8n + Docker + NGINX on Ubuntu 24.04 (OCI E2.1.Micro)
# Usage: sudo bash bootstrap_n8n.sh <domain> <email>
set -euxo pipefail

DOMAIN="${1:?Usage: sudo bash bootstrap_n8n.sh <domain> <email>}"
EMAIL="${2:?Usage: sudo bash bootstrap_n8n.sh <domain> <email>}"

# 1) Swap (1G) for small instances
if ! swapon --show | grep -q swapfile; then
  fallocate -l 1G /swapfile
  chmod 600 /swapfile
  mkswap /swapfile
  swapon /swapfile
  echo '/swapfile swap swap defaults 0 0' | tee -a /etc/fstab
fi

# 2) Docker Engine + compose
apt-get update -y
apt-get install -y ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
usermod -aG docker ubuntu || true

# 3) n8n via Docker Compose (localhost only; NGINX will proxy)
install -d -o ubuntu -g ubuntu /opt/n8n
cd /opt/n8n
if [ ! -f .env ]; then
  cat > .env <<EON8N
N8N_HOST=${DOMAIN}
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=$(openssl rand -base64 24 | tr -d '=+/')
GENERIC_TIMEZONE=UTC
EON8N
  chown ubuntu:ubuntu .env
fi

cat > docker-compose.yml <<'YAML'
services:
  n8n:
    image: n8nio/n8n:1.110.1
    container_name: n8n
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - WEBHOOK_URL=https://${N8N_HOST}/
      - N8N_EDITOR_BASE_URL=https://${N8N_HOST}/
    volumes:
      - ./data:/home/node/.n8n
YAML

mkdir -p /opt/n8n/data
chown -R 1000:1000 /opt/n8n/data

# start n8n
docker compose pull
docker compose up -d

# 4) NGINX
apt-get install -y nginx
mkdir -p /var/www/certbot
cat > /etc/nginx/sites-available/n8n.conf <<EONGINX
server {
    listen 80;
    server_name ${DOMAIN};

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://127.0.0.1:5678;
        client_max_body_size 20m;
        proxy_read_timeout 300;
    }
}
EONGINX
ln -sf /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/n8n.conf
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl restart nginx

echo ">>> DNS must point ${DOMAIN} to THIS VM's public IP."
echo ">>> Then install certbot and issue the cert:"
echo "sudo apt-get install -y certbot python3-certbot-nginx"
echo "sudo certbot --nginx -d ${DOMAIN} -m ${EMAIL} --agree-tos --redirect"
EOF

# Run it
sudo bash bootstrap_n8n.sh n8n.example.com you@example.com

# Issue the cert (after DNS resolves)
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d n8n.example.com -m you@example.com --agree-tos --redirect
```

> **Auto‑renewal**: Certbot installs a systemd timer that checks twice daily and renews when within 30 days of expiry. Verify via `systemctl status certbot.timer` and `sudo certbot renew --dry-run`.

---

## Detailed steps

### 1) Create the VM on OCI

* **Image:** Ubuntu 24.04 LTS (Canonical)
* **Shape:** `VM.Standard.E2.1.Micro` (Always Free)
* **Security list / NSG inbound:** TCP **22**, **80**, **443** from `0.0.0.0/0`
* **Route table:** default route (`0.0.0.0/0`) to **Internet Gateway**

### 2) Cloudflare DNS

* Add an **A** record for your subdomain to the VM public IP.
* During initial issuance, set **Proxy** = **DNS only (grey)** for simplicity.
* After cert is issued, switch back to **Proxied (orange)** and set **SSL/TLS = Full (strict)**.

### 3) Bootstrap the VM

Use the script in [Quick start](#quick-start). It installs Docker, Compose, n8n, and NGINX.

### 4) Issue TLS with Certbot

```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d n8n.example.com -m you@example.com --agree-tos --redirect
```

### 5) (Optional) Cloudflare proxy + real client IPs

**Recommended:** keep the DNS record proxied (orange). Add real‑IP config so NGINX sees the true client IP:

```bash
for i in $(curl -s https://www.cloudflare.com/ips-v4); do echo "set_real_ip_from $i;"; done | sudo tee /etc/nginx/conf.d/cloudflare-realip.conf
for i in $(curl -s https://www.cloudflare.com/ips-v6); do echo "set_real_ip_from $i;"; done | sudo tee -a /etc/nginx/conf.d/cloudflare-realip.conf
echo 'real_ip_header CF-Connecting-IP;' | sudo tee -a /etc/nginx/conf.d/cloudflare-realip.conf
sudo nginx -t && sudo systemctl reload nginx
```

**Optional hardening:** *Authenticated Origin Pulls* (allow only Cloudflare to reach your origin). Enable in Cloudflare, then on the server:

```bash
sudo curl -o /etc/ssl/certs/cloudflare_origin_ca.pem \
  https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem
# Add inside your HTTPS server block (Certbot will create it under sites-enabled):
# ssl_client_certificate /etc/ssl/certs/cloudflare_origin_ca.pem;
# ssl_verify_client on;
```

---

## Operations

### Upgrade n8n

```bash
cd /opt/n8n
# If staying on the pinned tag, pull updates and recreate
sudo docker compose pull
sudo docker compose up -d

# To move to a newer tag, edit docker-compose.yml (image: n8nio/n8n:<new>) then run the same two commands.
```

### Backup & restore

```bash
# Backup n8n data (workflows, creds) from the host
sudo tar czf /root/n8n-backup-$(date +%F).tgz -C /opt n8n/data

# Restore
sudo tar xzf /root/n8n-backup-YYYY-MM-DD.tgz -C /
sudo chown -R 1000:1000 /opt/n8n/data
cd /opt/n8n && sudo docker compose up -d
```

### Troubleshooting

**Certbot challenge fails (connection):**

* Use **grey cloud** during issuance, ensure port **80** is open on **OCI** and OS.
* Test locally: `echo ok | sudo tee /var/www/certbot/ping` then `curl -I http://<domain>/.well-known/acme-challenge/ping`.

**Permissions (EACCES writing /home/node/.n8n/config):**

```bash
cd /opt/n8n
sudo docker compose down
sudo chown -R 1000:1000 /opt/n8n/data
sudo docker compose up -d
```

**Docker networking breaks after custom iptables:**

```bash
# Prefer not to overwrite Docker's chains. Minimal reset:
sudo iptables -P FORWARD ACCEPT
sudo systemctl restart docker
```

---

## Security & what not to commit

* **Never commit secrets**: `.env`, `/opt/n8n/data/`, `/root/.cloudflare.ini`, logs with tokens.
* **Do not publish API tokens** (Cloudflare DNS, n8n credentials, OAuth secrets).
* **Public repo is OK** if you scrub secrets and IPs. Listing your domain is fine.
* Keep **port 80** open for HTTP‑01 renewals **or** switch to **DNS‑01** (no port 80 needed).

Add a `.gitignore` like:

```
# Secrets
.env
*.env
/root/.cloudflare.ini

# n8n data volume
/opt/n8n/data/
./data/

# Backups
*.tgz
/backups/

# Logs
*.log
/var/log/**
```

**DNS‑01 renewal (Cloudflare) — no port 80 required:**

```bash
sudo apt-get install -y python3-certbot-dns-cloudflare
sudo tee /root/.cloudflare.ini >/dev/null <<EOF
dns_cloudflare_api_token = <YOUR_TOKEN_WITH_Zone.DNS:Edit>
EOF
sudo chmod 600 /root/.cloudflare.ini
sudo certbot --nginx \
  --dns-cloudflare --dns-cloudflare-credentials /root/.cloudflare.ini \
  -d n8n.example.com -m you@example.com --agree-tos --keep-until-expiring
```

---

## Repo layout

```
.
├── README.md                # this file
├── docker-compose.yml       # compose for n8n (localhost:5678)
├── .env.example             # template (copy to /opt/n8n/.env on server)
├── scripts/
│   └── bootstrap_n8n.sh     # one-shot installer for the VM
├── nginx/
│   └── n8n.conf             # HTTP vhost (Certbot adds HTTPS)
└── .gitignore               # excludes secrets/data
```

### docker-compose.yml

```yaml
services:
  n8n:
    image: n8nio/n8n:1.110.1
    container_name: n8n
    restart: unless-stopped
    user: "1000:1000"
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - WEBHOOK_URL=https://${N8N_HOST}/
      - N8N_EDITOR_BASE_URL=https://${N8N_HOST}/
    volumes:
      - ./data:/home/node/.n8n
```

### .env.example

```env
N8N_HOST=n8n.example.com
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=change_me_now
GENERIC_TIMEZONE=UTC
```

### scripts/bootstrap\_n8n.sh

```bash
#!/usr/bin/env bash
set -euxo pipefail
DOMAIN="${1:?Usage: sudo bash bootstrap_n8n.sh <domain> <email>}"
EMAIL="${2:?Usage: sudo bash bootstrap_n8n.sh <domain> <email>}"
# (script body — same as Quick start, kept in scripts/)
```

### nginx/n8n.conf

```nginx
server {
    listen 80;
    server_name n8n.example.com;

    location /.well-known/acme-challenge/ { root /var/www/certbot; }

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_pass http://127.0.0.1:5678;
        client_max_body_size 20m;
        proxy_read_timeout 300;
    }
}
```

---


MIT — see `LICENSE`.
