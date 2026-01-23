# 🌐 Google Cloud Django Deployment

Production-ready guide to deploying a **Django application** on a
**Google Cloud VM** using:

-   Nginx (reverse proxy)
-   Gunicorn (WSGI server)
-   Supervisor (process manager)
-   Let's Encrypt (free SSL)

------------------------------------------------------------------------

## 📖 Overview

This repository provides step-by-step instructions to set up a secure
and scalable Django deployment on a Google Cloud virtual machine.\
It covers server preparation, application setup, domain configuration,
and HTTPS enablement.

------------------------------------------------------------------------

## 🧰 Tech Stack

-   **OS:** Ubuntu (Google Cloud VM)
-   **Backend:** Django + Gunicorn\
-   **Web Server:** Nginx\
-   **Process Manager:** Supervisor\
-   **Security:** Let's Encrypt SSL

------------------------------------------------------------------------

## ⚙️ Server Setup

``` bash
sudo apt-get update && sudo apt-get upgrade -y
```

------------------------------------------------------------------------

## 🔑 Git & SSH Configuration

``` bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

git config --global user.name "GitHub Username"
git config --global user.email "your_email@gmail.com"

cat ~/.ssh/id_rsa.pub
# Add this key to GitHub → Settings → SSH keys
```

------------------------------------------------------------------------

## 📦 Clone the Project

``` bash
mkdir ~/projects && cd ~/projects
git clone <YOUR_REPOSITORY_URL>
cd project_name
```

------------------------------------------------------------------------

## 🐍 Virtual Environment

``` bash
sudo apt install python3-venv -y
python3 -m venv env
source env/bin/activate
pip install -r requirements.txt
```

------------------------------------------------------------------------

## 🌍 Install Nginx

``` bash
sudo apt install nginx -y
sudo systemctl start nginx
sudo systemctl enable nginx
```

Create config:

``` bash
sudo rm /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/sites-enabled/default
```

``` nginx
server {
    listen 80;
    server_name your_server_ip;

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:8000;
    }
}
```

Restart:

``` bash
sudo systemctl restart nginx
```

------------------------------------------------------------------------

## 🔫 Gunicorn

``` bash
pip install gunicorn
/home/username/projects/projectname/env/bin/gunicorn core.wsgi:application --bind 127.0.0.1:8000 --workers 3
```

------------------------------------------------------------------------

## ♻️ Supervisor

``` bash
sudo apt install supervisor -y
sudo systemctl start supervisor
```

Create config:

``` bash
sudo nano /etc/supervisor/conf.d/projectname_gunicorn.conf
```

``` ini
[program:projectname_gunicorn]
user=root
directory=/home/username/projects/projectname
command=/home/username/projects/projectname/env/bin/gunicorn core.wsgi:application --bind 127.0.0.1:8000 --workers 3
autostart=true
autorestart=true
stdout_logfile=/var/log/projectname/gunicorn.log
stderr_logfile=/var/log/projectname/gunicorn.err.log
```

``` bash
sudo mkdir -p /var/log/projectname
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start projectname_gunicorn
```

------------------------------------------------------------------------

## 🎨 Static Files

``` bash
python manage.py collectstatic
```

``` python
from django.contrib.staticfiles.urls import staticfiles_urlpatterns
urlpatterns += staticfiles_urlpatterns()
```

Restart services:

``` bash
sudo systemctl restart nginx
sudo supervisorctl restart projectname_gunicorn
```

------------------------------------------------------------------------

## 🌐 Domain & DNS

-   Create an **A record**
-   Point it to your Google Cloud VM external IP

Clear DNS cache (Windows):

``` bash
ipconfig /flushdns
```

------------------------------------------------------------------------

## 🔒 SSL with Let's Encrypt

``` bash
sudo apt install letsencrypt python3-certbot-nginx -y
sudo certbot certonly --manual --agree-tos --preferred-challenges dns -d domain.com -d *.domain.com
```

------------------------------------------------------------------------

## 🔐 HTTPS Nginx Config

``` nginx
server {
    listen 80;
    server_name domain.com;
    rewrite ^ https://$server_name$request_uri? permanent;
}

server {
    listen 443 ssl;
    server_name domain.com;

    ssl_certificate /etc/letsencrypt/live/domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.com/privkey.pem;

    location / {
        include proxy_params;
        proxy_pass http://127.0.0.1:8000;
    }
}
```

``` bash
sudo systemctl restart nginx
```

------------------------------------------------------------------------

## ✅ Deployment Complete

Your Django project is now running with:

-   Production WSGI server\
-   Reverse proxy\
-   Automatic restarts\
-   HTTPS encryption

------------------------------------------------------------------------

## 📄 License

This project is licensed under the MIT License.

------------------------------------------------------------------------

## 🤝 Contributing

Pull requests are welcome. For major changes, please open an issue first
to discuss what you would like to change.
