# Deploy Multiple Django Projects on One Server with URL Prefixes

This guide shows how to deploy **two separate Django projects** on the **same server and domain**, each accessible via unique URL prefixes `/project1/` and `/project2/`.

---

## 1. Directory Structure Example

```bash
/home/username/
├── project1/
│   ├── mysite/
│   ├── venv/
│   └── static/
├── project2/
│   ├── mysite/
│   ├── venv/
│   └── static/

```

##2. Django settings.py Configuration
Project 1
In project1/mysite/settings.py:
```python
FORCE_SCRIPT_NAME = '/project1'
STATIC_URL = '/project1/static/'
STATIC_ROOT = '/home/username/project1/static/'

ALLOWED_HOSTS = ['example.com']
```
Project 2
In project2/mysite/settings.py:
```python
FORCE_SCRIPT_NAME = '/project2'
STATIC_URL = '/project2/static/'
STATIC_ROOT = '/home/username/project2/static/'

ALLOWED_HOSTS = ['example.com']
``` 
##3. Collect Static Files
Run this for each project to collect static files:
```python
cd /home/username/project1/mysite
source ../venv/bin/activate
python manage.py collectstatic --noinput
deactivate

cd /home/username/project2/mysite
source ../venv/bin/activate
python manage.py collectstatic --noinput
deactivate
``` 

##4. Create Gunicorn Systemd Service Files
/etc/systemd/system/project1.service

[Unit]
Description=Gunicorn Project1
After=network.target
```ini
[Service]
User=www-data
Group=www-data
WorkingDirectory=/home/username/project1/mysite
ExecStart=/home/username/project1/venv/bin/gunicorn mysite.wsgi:application --bind unix:/run/project1.sock
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=gunicorn-project1

[Install]
WantedBy=multi-user.target
```

/etc/systemd/system/project2.service
```ini
[Unit]
Description=Gunicorn Project2
After=network.target

[Service]
User=www-data
Group=www-data
WorkingDirectory=/home/username/project2/mysite
ExecStart=/home/username/project2/venv/bin/gunicorn mysite.wsgi:application --bind unix:/run/project2.sock
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=gunicorn-project2

[Install]
WantedBy=multi-user.target
```

Reload systemd and start the services:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now project1.service
sudo systemctl enable --now project2.service
```

##5. Configure Nginx
Create /etc/nginx/sites-available/multiple-projects with this content:

```nginx
server {
    listen 80;
    server_name example.com;

    access_log /var/log/nginx/project1_access.log;
    error_log /var/log/nginx/project1_error.log;

    location /project1/static/ {
        alias /home/username/project1/static/;
    }

    location /project2/static/ {
        alias /home/username/project2/static/;
    }

    location /project1/ {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_pass http://unix:/run/project1.sock;
    }

    location /project2/ {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
        proxy_pass http://unix:/run/project2.sock;
    }
}
```

###Enable the site and reload Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/multiple-projects /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```



