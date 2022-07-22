# 3DLOOK test task

## Automatic deployment
### Dependencies
- python3
- pip
- pipenv
- VirtualBox
- Vagrant

### Prepare python environment
```bash
pipenv install -r requirements.txt
```
### Prepare inventory file
Make copy of *inventory.yml.template* and rename it to *inventory.yml*.
```bash
cp inventory.yml.template inventory.yml
```
Open a text editor and fill in the given directives.
```bash
vi inventory.yml
```
Example:
```yml
all:
  hosts:
    server:
      ansible_host: 192.168.1.24
      ansible_user: admin
      ansible_ssh_private_key_file: ~/.ssh/server.pem
```

### Prepare vars_file
Make copy of *vars_file.yml.template* and rename it to *vars_file.yml*.
```bash
cp vars_file.yml.template vars_file.yml
```
Open a text editor and fill in the given directives.
```bash
vi vars_file.yml
```
Example:
```yml
db_name: django_project
db_user: django_user
db_passwd: qwerty1234
pgadmin_admin_email: test@te.st
pgadmin_admin_passwd: qwerty123
domain_name: jofurytestsowntest.tk
django_project_name: django_project
django_key: 5#sg3$!KL!yaFyh$msSa9&JW6pc1M2vcm2S&%9#Sz$M9!b6eFt*
```

### Deployment
Enter to pipenv shell
```bash
pipenv shell
```
Apply playbook
```bash
ansible-playbook playbook.yml
```
For local deployment via Vagrant
```bash
vagrant up --provision
```


## Manual deployment
### Base system configure
Update system and install the necessary packages.
```bash
sudo apt update -y && sudo apt upgrade -y && sudo apt install -y \
	python3-pip \
	python3-dev \
	virtualenv \
	postgresql \
	postgresql-contrib \
	nginx \
	curl \
	ufw \
	python3-certbot \
	python3-certbot-nginx
```
Configure firewall.
```bash
sudo ufw allow 22/tcp # ssh
sudo ufw allow 80/tcp # http
sudo ufw allow 443/tcp # https
sudo ufw enable
```

### Create PostgreSQL database and user
Log into postgress interactive session.
```bash
sudo -u postgres psql
```
In psql shell, we will create a database, create a user and apply a few settings as recommended by the documentation and let the new user manage this db.  
***Replace \$PASSWORD with your own value***.
```sql
CREATE DATABASE django;
CREATE USER django_user WITH PASSWORD '$PASSWORD';
ALTER ROLE django_user SET client_encoding TO 'utf8';
ALTER ROLE django_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE django_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE django TO django_user;
\q
```
Make an automatic daily backup of the database at 00:59.  
Create dir for backups.
```bash
sudo mkdir /var/postgres_backup/
```
Change dir owner.
```bash
chown postgres.postgres -R /var/postgres_backup
```
Add job to crontab. Run

```bash
sudo crontab -u postgres -e 
```
and add this line:
``` bash
59 0 * * * pg_basebackup -Ft -z -D /var/postgres_backup/$(date +\%d_\%m_\%Y)/
```
### Install pgAdmin.
Create virtualenv for pgAdmin.
```bash
virtualenv ~/pgadmin
```
Enter to pgAdmin virtualenv.
```bash
source ~/pgadmin/bin/activate
```
Install pgadmin and uwsgi.
```bash
pip install uwsgi pgadmin4
```
Create necessary for pgAdmin dirs and set permissions.
```bash
sudo mkdir -p /var/lib/pgadmin/{sessions,storage}
sudo mkdir /var/log/pgadmin
sudo chown -R $USER /var/lib/pgadmin
sudo chown -R $USER /var/log/pgadmin
```
Configure pgAdmin. In *~/pgadmin/lib/python3.9/site-packages/pgadmin4/config_local.py* add these lines:
```
LOG_FILE = '/var/log/pgadmin/pgadmin.log'
SQLITE_PATH = '/var/lib/pgadmin/pgadmin.db'
SESSION_DB_PATH = '/var/lib/pgadmin/sessions'
STORAGE_DIR = '/var/lib/pgadmin/storage'
SERVER_MODE = True
```
Create account for pgAdmin. Run:
```bash
python3 ~/pgadmin/lib/python3.9/site-packages/pgadmin4/setup.py
```
and follow the instructions.  
Leave virtualenv.
```bash
deactivate
```
### Install Django
Create virtualenv for Django.
```bash
virtualenv ~/django
```
Enter to Django virtualenv.
```bash
source ~/django/bin/activate
```
Install django, psycopg2-binary and uwsgi.
```bash
pip install django uwsgi psycopg2-binary
```
Create Django project.
```bash
django-admin startproject django_project ~/django
```
Configure Django project.  
Set the ALLOWED_HOST directive.
```py
ALLOWED_HOSTS = ['localhost', '$DOMAIN_OR_IP', 'www.$DOMAIN_OR_IP']
```
*Replace $DOMAIN_OR_IP with domain or ip address of your server.*  
\
Set PostgreSQL as default DB. In *~/django/django_project/settings.py* edit *DATABASES* directive.  
***Replace \$PASSWORD with the value which you have set in your database.***

```py
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'django',
        'USER': 'django_user',
        'PASSWORD': '$PASSWORD',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```
Set dir with static content. Add to file this line:
```py
STATIC_ROOT = BASE_DIR / 'static/'
```
Migrate initial db schema to postgres.
```bash
python3 ~/django/manage.py makemigrations
python3 ~/django/manage.py migrate
```
Collect all of the static content into directory which we specified above.
```bash
python3 ~/django/manage.py collectstatic
```
Leave virtualenv.
```bash
deactivate
```
### Configure uWSGI
Create dirs for uWSGI configs.
```bash
sudo mkdir -p /etc/uwsgi
```
Create config for Django.  
*/etc/uwsgi/django.ini*
```ini
[uwsgi]
project = /home/$USER_NAME/django
project_name = django_project
user = $USER_NAME

chdir = %(project)
module = %(project_name).wsgi:application

master = true
processes = 5

socket = /tmp/%(project_name).sock
chown-socket = %(user):www-data
chmod-socket = 660
vacuum = true
```
*Replace $USER_NAME with your username.*  
\
Create systemd-service for Django project.  
*/etc/systemd/system/django_uwsgi.service* 
```ini
[Unit]
Description=Django on uWSGI
Requires=network.target
After=network.target

[Service]
User=$USER_NAME
Group=www-data
WorkingDirectory=/home/$USER_NAME/django
Environment="PATH=/home/$USER_NAME/django/bin"
ExecStart= /home/$USER_NAME/django/bin/uwsgi --ini /etc/uwsgi/django.ini
Restart=on-failure
RestartSec=10
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all
StandardError=syslog

[Install]
WantedBy=multi-user.target
```
*Replace $USER_NAME with your username.*  
\
Start django_uwsgi service.
```bash
sudo systemctl enable --now django_uwsgi.service
```
Create config for pgAdmin.  
*/etc/uwsgi/pgadmin.ini*
```ini
[uwsgi]
project = /home/$USER_NAME/pgadmin
project_name = pgadmin
user = $USER_NAME

chdir = %(project)/lib/python3.9/site-packages/pgadmin4/
mount = /pgadmin4=pgAdmin4:app

master = true
processes = 1
threads = 25

socket = /tmp/%(project_name).sock
chown-socket = %(user):www-data
chmod-socket = 660
manage-script-name = true
vacuum = true
```
*Replace $USER_NAME with your username.*  
\
Create systemd-service for pgAdmin.  
*/etc/systemd/system/pgadmin_uwsgi.service* 
```ini
[Unit]
Description=pgadmin4 on uWSGI
Requires=network.target
After=network.target

[Service]
User=$USER_NAME
Group=www-data
WorkingDirectory=/home/$USER_NAME/pgadmin/lib/python3.9/site-packages/pgadmin4
Environment="PATH=/home/$USER_NAME/pgadmin/bin"
ExecStart= /home/$USER_NAME/pgadmin/bin/uwsgi --ini /etc/uwsgi/pgadmin.ini
Restart=on-failure
RestartSec=10
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all
StandardError=syslog

[Install]
WantedBy=multi-user.target
```
*Replace $USER_NAME with your username.*  
\
Start pgadmin_uwsgi service.
```bash
sudo systemctl enable --now pgadmin_uwsgi.service
```
### Configure nginx
In */etc/nginx/sites-available/test_project.conf* add:
```
upstream django {
    server unix:///tmp/django_project.sock;
}

upstream pgadmin {
    server unix:///tmp/pgadmin.sock;
}

server {
    listen      80;
    server_name $DOMAIN_OR_IP www.$DOMAIN_OR_IP;
    charset     utf-8;

    client_max_body_size 75M;

    location /static {
        alias /home/$USER_NAME/django/static;
    }

    location / {
        include     /etc/nginx/uwsgi_params;
        uwsgi_pass  django;
    }

    location = /pgadmin4 { rewrite ^ /pgadmin4/; }
    location /pgadmin4 { try_files $uri @pgadmin4; }
    location @pgadmin4 {
	include /etc/nginx/uwsgi_params;
	uwsgi_pass pgadmin;
    }
}
```
*Replace $DOMAIN_NAME with domain name of your server and $USER_NAME with your username.*  
\
Enbale this config.
```bash
sudo ln -s /etc/nginx/sites-available/test_project.conf /etc/nginx/sites-enabled/
```
Gen SSL cert from Let's Encrypt.
```bash
sudo certbot --nginx --agree-tos --register-unsafely-without-email -n -d $DOMAIN_NAME -d www.$DOMAIN_NAME
```
*Replace $DOMAIN_NAME with domain name of your server.*  
\
Verify that certificate auto-renewal works.
```bash
systemctl status certbot.timer
```
```
● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since Wed 2022-07-20 23:17:41 UTC; 28min ago
    Trigger: Thu 2022-07-21 08:53:46 UTC; 9h left
   Triggers: ● certbot.service
```
