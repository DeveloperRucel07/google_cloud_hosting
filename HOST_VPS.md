# VPS Hosting Guide — Website & Subdomains

> A complete guide to hosting a website and multiple subdomains on a Linux VPS (Ubuntu 24).

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Server Setup](#server-setup)
3. [DNS Configuration](#dns-configuration)
4. [Install Nginx](#install-nginx)
5. [Deploy Your Apps](#deploy-your-apps)
6. [Nginx Configuration](#nginx-configuration)
7. [SSL Certificates](#ssl-certificates)
8. [Running an API with PM2](#running-an-api-with-pm2)
9. [Deploy Script](#deploy-script)
10. [Troubleshooting](#troubleshooting)

---

## 1. Prerequisites

Before you start, make sure you have:

- A VPS running Ubuntu 22.04 or 24.04
- A registered domain name (e.g. `yourdomain.com`)
- SSH access to your VPS
- Your projects hosted on GitHub

---

## 2. Server Setup

SSH into your VPS:

```bash
ssh user@YOUR.VPS.IP
```

Update the system and install essential tools:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install git curl -y
```

Install Node.js using NVM (recommended):

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
source ~/.bashrc
nvm install --lts
```

Verify installation:

```bash
node -v
npm -v
```

---

## 3. DNS Configuration

Log in to your domain registrar (e.g. IONOS) and add the following **A records**, all pointing to your VPS IP address:

| Type | Host       | Points To       | TTL  |
|------|------------|-----------------|------|
| A    | `@`        | `YOUR.VPS.IP`   | 3600 |
| A    | `www`      | `YOUR.VPS.IP`   | 3600 |
| A    | `api`      | `YOUR.VPS.IP`   | 3600 |
| A    | `app`      | `YOUR.VPS.IP`   | 3600 |
| A    | `react`    | `YOUR.VPS.IP`   | 3600 |
| A    | `static`   | `YOUR.VPS.IP`   | 3600 |

> **Note:** DNS propagation can take up to 48 hours. Check progress at [dnschecker.org](https://dnschecker.org).

> **Tip:** You do NOT need to register or activate subdomains separately. Adding the A record is all it takes to create a subdomain.

---

## 4. Install Nginx

```bash
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

Open firewall ports:

```bash
sudo ufw allow 80
sudo ufw allow 443
sudo ufw enable
```

Verify Nginx is running:

```bash
sudo systemctl status nginx
```

---

## 5. Deploy Your Apps

### Create Web Root Directories

```bash
sudo mkdir -p /var/www/yourdomain.com
sudo mkdir -p /var/www/app.yourdomain.com
sudo mkdir -p /var/www/react.yourdomain.com
sudo mkdir -p /var/www/static.yourdomain.com
```

### Clone and Build Each App

**Angular App (Main Domain):**

```bash
cd /var/www/yourdomain.com
sudo git clone https://github.com/yourusername/your-angular-app .
npm install
npm run build --configuration production
# Build output: dist/your-app-name/browser/
```

**Angular App (Subdomain):**

```bash
cd /var/www/app.yourdomain.com
sudo git clone https://github.com/yourusername/your-angular-app2 .
npm install
npm run build --configuration production
```

**React App:**

```bash
cd /var/www/react.yourdomain.com
sudo git clone https://github.com/yourusername/your-react-app .
npm install
npm run build
# Build output: build/
```

**HTML/CSS/JS App (no build needed):**

```bash
sudo git clone https://github.com/yourusername/your-static-app /var/www/static.yourdomain.com
```

**API:**

```bash
sudo git clone https://github.com/yourusername/your-api /var/www/api
cd /var/www/api
npm install
```

### Fix File Permissions

```bash
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/
```

---

## 6. Nginx Configuration

Create one config file per domain/subdomain inside `/etc/nginx/sites-available/`.

### Main Domain — Angular App

`/etc/nginx/sites-available/yourdomain.com`

```nginx
# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://$host$request_uri;
}

# Serve HTTPS
server {
    listen 443 ssl;
    server_name yourdomain.com www.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    root /var/www/yourdomain.com/dist/your-app-name/browser;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Angular App — Subdomain

`/etc/nginx/sites-available/app.yourdomain.com`

```nginx
server {
    listen 80;
    server_name app.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    root /var/www/app.yourdomain.com/dist/your-app2-name/browser;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### React App — Subdomain

`/etc/nginx/sites-available/react.yourdomain.com`

```nginx
server {
    listen 80;
    server_name react.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name react.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    root /var/www/react.yourdomain.com/build;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### HTML/CSS/JS App — Subdomain

`/etc/nginx/sites-available/static.yourdomain.com`

```nginx
server {
    listen 80;
    server_name static.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name static.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    root /var/www/static.yourdomain.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### API — Subdomain

`/etc/nginx/sites-available/api.yourdomain.com`

```nginx
server {
    listen 80;
    server_name api.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;  # change to your API port
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Enable All Sites

```bash
sudo ln -s /etc/nginx/sites-available/yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/app.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/react.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/static.yourdomain.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/api.yourdomain.com /etc/nginx/sites-enabled/

# Remove default config to avoid conflicts
sudo rm /etc/nginx/sites-enabled/default

# Test and reload
sudo nginx -t
sudo systemctl reload nginx
```

---

## 7. SSL Certificates

Install Certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Issue certificates for all domains at once:

```bash
sudo certbot --nginx \
  -d yourdomain.com \
  -d www.yourdomain.com \
  -d app.yourdomain.com \
  -d react.yourdomain.com \
  -d static.yourdomain.com \
  -d api.yourdomain.com
```

Certbot auto-renews certificates. Verify auto-renewal:

```bash
sudo certbot renew --dry-run
## add cronjob for automacally renew every month
sudo crontab -e
### an then add this line to the cron
0 3 1 * * /usr/bin/certbot renew --quiet --deploy-hook "systemctl reload nginx"
```

Check existing certificates:

```bash
sudo certbot certificates
```

---

## 8. Running an API with PM2

PM2 is a process manager that keeps your API running in the background and restarts it on crashes or server reboots.

```bash
# Install PM2 globally
npm install -g pm2

# Start your API
cd /var/www/api
pm2 start index.js --name "api"   # replace index.js with your entry file

# Save and enable auto-start on reboot
pm2 startup
pm2 save
```

Useful PM2 commands:

```bash
pm2 list              # list all running processes
pm2 logs api          # view API logs
pm2 restart api       # restart the API
pm2 stop api          # stop the API
pm2 delete api        # remove from PM2
```

---

## 9. Deploy Script

Create a shell script to update each app quickly after pushing to GitHub.

**Angular app deploy script** (`/var/www/deploy-angular.sh`):

```bash
#!/bin/bash
echo "Deploying Angular app..."
cd /var/www/yourdomain.com
git pull
npm install
npm run build --configuration production
echo "✅ Angular app deployed!"
```

**API deploy script** (`/var/www/deploy-api.sh`):

```bash
#!/bin/bash
echo "Deploying API..."
cd /var/www/api
git pull
npm install
pm2 restart api
echo "✅ API deployed!"
```

Make scripts executable:

```bash
chmod +x /var/www/deploy-angular.sh
chmod +x /var/www/deploy-api.sh
```

Run anytime you push new code:

```bash
./deploy-angular.sh
./deploy-api.sh
```

---

## 10. Troubleshooting

### Nginx won't start or reload

```bash
sudo nginx -t          # check for config errors
sudo journalctl -xe    # view detailed logs
```

### Conflicting server names

If you see `conflicting server name` warnings:

```bash
# Find duplicate domain definitions
grep -r "yourdomain.com" /etc/nginx/

# Remove the default config
sudo rm /etc/nginx/sites-enabled/default

sudo nginx -t
sudo systemctl reload nginx
```

### Website shows nothing / 404

```bash
# Check files exist
ls -la /var/www/yourdomain.com/

# Check permissions
sudo chown -R www-data:www-data /var/www/
sudo chmod -R 755 /var/www/

# Check error logs
sudo tail -f /var/log/nginx/error.log
```

### Check firewall

```bash
sudo ufw status
# Should show 80 and 443 as ALLOWED
```

### SSL certificate issues

```bash
sudo certbot certificates   # check cert paths and expiry
sudo certbot renew          # manually renew
```

---

## Quick Reference

| Domain | App Type | Root Path |
|--------|----------|-----------|
| `yourdomain.com` | Angular | `/var/www/yourdomain.com/dist/app/browser` |
| `app.yourdomain.com` | Angular | `/var/www/app.yourdomain.com/dist/app2/browser` |
| `react.yourdomain.com` | React | `/var/www/react.yourdomain.com/build` |
| `static.yourdomain.com` | HTML/CSS/JS | `/var/www/static.yourdomain.com` |
| `api.yourdomain.com` | API (proxy) | `localhost:3000` |

---
