# 🚀 Django Production Deployment Guide (Ubuntu + Gunicorn + Nginx + SQLite + Cron + IST Timezone)

---

## 📦 1. Pre-Deployment (Local Machine)

- Push your code to GitHub
- Ensure `requirements.txt` is updated
- In `settings.py`:
  ```python
  DEBUG = False
  ALLOWED_HOSTS = ['your_server_ip']

  STATIC_URL = '/static/'
  STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')

  TIME_ZONE = 'Asia/Kolkata'   # ✅ IST Timezone
  USE_TZ = True                # ✅ Enable timezone support
  ```
- Run:
  ```bash
  python manage.py collectstatic
  ```

---

## 💻 2. Connect to Server

```bash
ssh user@your_server_ip
```

---

## 🛠️ 3. Server Setup

```bash
sudo apt update
sudo apt install python3-pip python3-dev python3-venv nginx
```

---

## 📁 4. Project Setup

```bash
cd ~
mkdir project_name
cd project_name
python3 -m venv venv
source venv/bin/activate
```

---

## 📥 5. Clone Project & Install Dependencies

```bash
git clone https://github.com/your/repo.git -b branchname
cd your_project
pip install -r requirements.txt
```

---

## 🔄 6. Migrations & Superuser

```bash
python manage.py makemigrations
python manage.py migrate
python manage.py createsuperuser
```

---

## 🔧 7. Test Local Server

```bash
sudo ufw allow 8000
python manage.py runserver 0.0.0.0:8000
```

Test: `http://your_server_ip:8000`

---

## 🔥 8. Gunicorn Setup

### Test it first:

```bash
gunicorn --bind 0.0.0.0:8000 your_project.wsgi
```

---

### Create Gunicorn Socket

```bash
sudo vim /etc/systemd/system/gunicorn.socket
```

Paste:

```ini
[Unit]
Description=gunicorn socket

[Socket]
ListenStream=/run/gunicorn.sock

[Install]
WantedBy=sockets.target
```

---

### Create Gunicorn Service

```bash
sudo vim /etc/systemd/system/gunicorn.service
```

Paste:

```ini
[Unit]
Description=gunicorn daemon
Requires=gunicorn.socket
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/user/project_name/your_project
ExecStart=/home/user/project_name/venv/bin/gunicorn \
          --access-logfile - \
          --workers 3 \
          --bind unix:/run/gunicorn.sock \
          your_project.wsgi:application
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```

---

## ▶️ 9. Start Gunicorn

```bash
sudo systemctl start gunicorn.socket
sudo systemctl enable gunicorn.socket
```

---

## 🌐 10. Nginx Configuration

```bash
sudo vim /etc/nginx/sites-available/your_project
```

Paste:

```nginx
server {
    listen 80;
    server_name your_server_ip;

    location = /favicon.ico { access_log off; log_not_found off; }

    location /static/ {
        alias /home/user/project_name/your_project/staticfiles/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/run/gunicorn.sock;
    }
}
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/your_project /etc/nginx/sites-enabled/
```

---

## 🔁 11. Restart Services

```bash
sudo systemctl daemon-reload
sudo systemctl restart nginx
sudo systemctl restart gunicorn
sudo systemctl enable gunicorn
```

---

## 🕐 12. Cron Job Setup

### 🧾 Create Shell Script

```bash
sudo nano /home/user/project_name/your_project/run_tasks.sh
```

Paste:

```bash
#!/bin/bash
cd /home/user/project_name/your_project
/home/user/project_name/venv/bin/python script1.py &
/home/user/project_name/venv/bin/python script2.py &
```

Make executable:

```bash
chmod +x /home/user/project_name/your_project/run_tasks.sh
```

---

### 🗓️ Add to Crontab

```bash
crontab -e
```

Paste:

```bash
* * * * * /bin/bash /home/user/project_name/your_project/run_tasks.sh >> /home/user/cron.log 2>&1
```

(Optional: No logs)

```bash
* * * * * /bin/bash /home/user/project_name/your_project/run_tasks.sh > /dev/null 2>&1 &
```

---

### 🛠️ Manage Cron

```bash
crontab -l          # View
crontab -e          # Edit
pgrep -a python     # List running python scripts
kill -9 <PID>       # Kill a job
```

---

## ⏰ 13. Set Server Timezone (IST)

```bash
sudo timedatectl set-timezone Asia/Kolkata
timedatectl  # to confirm
```

---

## 📄 14. Logs with journalctl

```bash
sudo journalctl -u gunicorn.service        # Full logs
sudo journalctl -u gunicorn.service -f     # Live logs
sudo tail -f /var/log/nginx/error.log      # Nginx errors
sudo tail -f /var/log/nginx/access.log     # Nginx access
journalctl -u gunicorn.service --since today --no-pager -r  # View Only Logs From Today
journalctl -u gunicorn.service -f --no-pager  # -f: Follows logs in real time  --no-pager: Again, prevents the pager
journalctl -u gunicorn.service --no-pager -r | grep "ERROR"  # search for errors

```

---

## ✅ DONE — App is Live at http://your_server_ip
