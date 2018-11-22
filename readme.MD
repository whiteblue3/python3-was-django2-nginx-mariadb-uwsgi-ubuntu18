Pre-Requirements
================

```
Ubuntu >= 18.04 LTS
Python >= 3.6.6
DJango >= 2.1.2
uWSGI
NginX

AWS EC2
AWS S3 (Needs static and media directory under your s3 bucket)
AWS RDS (option, tested mysql and mariadb)
```


Setting for Python3 and DJango2
===============================

<p>1. install pre requirement packages</p>

```
sudo apt-get install python-dev python-setuptools python-pip python3-pip python-django-common python3-django
pip install --upgrade pip
```

<p>2. check /srv directory is exist and create directory if no exist</p>

```
sudo mkdir /srv
```

<p>3. change ownership to user</p>

```
sudo chown -R ubuntu.ubuntu /srv
```

<p>4. create logs directory and log file</p>

```
cd /srv
mkdir logs
mkdir logs/django
touch logs/django/log
```

This directory use for django log


NginX Configuration
===================

<p>1. install server nginx</p>

```
sudo apt-get install nginx nginx-extras
```

nignx-extras need for hide or change "Server" headers you want

<p>2. configure /etc/nginx/nginx.conf<p>

- set server_tokens off
```
http {
...
	server_tokens off;
...
}
```

This option is uses for security issue

- set TLS version
```
http {
...
	ssl_protocols TLSv1.2;
...
}
```

This option is uses for security issue

- set security headers
```
http {
...
	# Security
        fastcgi_hide_header X-Powered-By;
        fastcgi_hide_header X-Pingback;
        fastcgi_hide_header Link;
        proxy_hide_header X-Powered-By;
        proxy_hide_header X-Pingback;
        proxy_hide_header X-Link;
        more_set_headers "Server: server"; # You can change your HTTP "Server" header anything you want. DO NOT set this field to nginx, apache or anyother server name.

        set_real_ip_from    0.0.0.0/0; # Allow any other client ip to logging
        real_ip_header      X-Forwarded-For; # Set access log to real ip
...
}
```

<p>3. configure /etc/nginx/sites-available/default :</p>

- you can redirect http to https for security issue
```
server {
        listen 80;
        listen [::]:80;

        return 301 https://$host$request_uri?;
}
```

- setup default information
```
server {
...
	listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;
        charset utf-8;

        # SSL configuration
        # This is optional for https
        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 default_server ipv6only=on;
...
}
```

- security header setup
```
server {
...
	add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
        add_header X-Frame-Options "DENY"; # or SAMEORIGIN is also safe.
        add_header X-Content-Type-Options "nosniff";
        add_header Cache-Control "no-cache";
        add_header X-XSS-Protection "1; mode=block";
...
}
```

This options are also included in django settings

/srv/was/conf/settings/base.py :

```
SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https') #for HTTPS
SECURE_SSL_REDIRECT = True #for HTTPS
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_HSTS_SECONDS = 63072000
SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_HSTS_PRELOAD = True
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
X_FRAME_OPTIONS = 'SAMEORIGIN'
```

If you want to use django setup. you must comment above setup

```
server {
...
	#add_header Strict-Transport-Security "max-age=63072000; includeSubdomains; preload";
        #add_header X-Frame-Options "DENY"; # or SAMEORIGIN is also safe.
        #add_header X-Content-Type-Options "nosniff";
        add_header Cache-Control "no-cache";
        #add_header X-XSS-Protection "1; mode=block";
...
}
```

Also if you want to use nginx setup. you must comment was/conf/settings/base.py

```
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
#SECURE_HSTS_SECONDS = 63072000
#SECURE_CONTENT_TYPE_NOSNIFF = True
SECURE_BROWSER_XSS_FILTER = True
#SECURE_HSTS_PRELOAD = True
#SECURE_HSTS_INCLUDE_SUBDOMAINS = True
#X_FRAME_OPTIONS = 'SAMEORIGIN'
```

- additionally you can use your custom ssl certificate
```
server {
...
	ssl_certificate /path/to/signed_cert_plus_intermediates; # your certificate path
	ssl_certificate_key /path/to/private_key; # your private certificate path
	ssl_session_timeout 1d;
	ssl_session_cache shared:SSL:50m;
	ssl_session_tickets off;
	
	ssl_protocols TLSv1.2;
	ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
	ssl_prefer_server_ciphers on;
	
	ssl_stapling on;
	ssl_stapling_verify on;
	
	## verify chain of trust of OCSP response using Root CA and Intermediate certs
	ssl_trusted_certificate /path/to/root_CA_cert_plus_intermediates;

	resolver <IP DNS resolver>;
...
}
```


- keeyalive timeout setup
```
server {
...
	keepalive_timeout 5;
...
}
```

- you can change your server name
```
server {
...
	server_name _;
...
}
```

This is prevent to hacker cannot acquire server type like apache or nginx, lighttpd...

- allow and enable your python was
```
server {
...
	location / {
		# This is not safe. you should create custom error pages
		# you must disable it to use django include function
                #try_files $uri $uri/ =404;

		# Setup this for your environment
                limit_except GET POST {
                        deny all;
                }

		uwsgi_pass unix:///tmp/was.sock; # your was socket
                include uwsgi_params;

                proxy_set_header REMOTE_ADDR $remote_addr;
        }
...
}
```


uWSGI Configurations
====================

<p>1. Install requirements packages</p>

```
sudo apt-get install uwsgi uwsgi-plugin-python3
```

<p>2. create uwsgi ini and unix socket</p>

```
sudo mkdir /etc/uwsgi/sites
sudo touch /etc/uwsgi/sites/was.ini
touch /tmp/was.sock
chmod 666 /tmp/was.sock
```

/etc/uwsgi/sites/was.ini :
```
[uwsgi]
vhost = false
plugin = python
chmod-socket = 666
socket = /tmp/was.sock
master = true
enable-threads = true
process = 2
wsgi-file = /srv/was/conf/wsgi.py
chdir = /srv/was
module = was.wsgi:application
```

wsgi-file is under anywhere you want path. input your wsgi.py path. default is /srv/was/conf/wsgi.py

chdir is your project path. default is /srv/was

module is your was wsgi application name. default is was.wsgi:application

<p>3. install uwsgi via pip3 and enable it</p>

```
sudo -H pip3 install uwsgi # or no need this line
sudo systemctl start uwsgi
sudo systemctl enable uwsgi
```

<p>4. create uwsgi.service</p>

/etc/systemd/system/uwsgi.service :
```
[Unit]
Description=uWSGI Emperor service

[Service]
ExecStart=/usr/bin/uwsgi_python36 --master --die-on-term --emperor /etc/uwsgi/sites
Restart=on-failure
KillSignal=SIGQUIT
Type=notify
NotifyAccess=all
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

--die-on-term : if your uwsgi is killed. this option is auto-start uwsgi

--emperor : your was.ini directory path


DJango WAS Project Installations
================================

un-archive python3-was-django2-nginx-mariadb-uwsgi-ubuntu18.tar to under /srv

you will show was, logs directory is under python3-was-django2-nginx-mariadb-uwsgi-ubuntu18

please move was and logs directory to /srv

<p>1. install awscli</p>

```
sudo apt-get install awscli
```

<p>2. configure awscli</p>

```
aws configure

AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [ap-northeast-2]: <-- This is a Seoul region
Default output format [json]: 
```

You must create access key first on you aws console

<p>3. edit was/.config_secret/settings_common.py</p>

- django secret key
    -> You can generate secret key at here : https://www.miniwebtool.com/django-secret-key-generator/
- aws access key
- aws secret access key

<p>4. edit was/.config_secret/settings_dev.py and was/.config_secret/settings_prod.py</p>

- edit allowed host
- edit db config
- aws s3 bucket name

<p>5. finally config settings/</p>

configure settings/base.py, settings/dev.py, settings/prod.py accept your environment

if you are not want to service https, comment this lines on settings/base.py

settings/base.py :

```
#SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTO', 'https')
#SECURE_SSL_REDIRECT = True
```

This will disable https


<p>6. test associate with aws s3</p>

First of all, your s3 bucket needs static and media directory

```
python3 manage.py collectstatic
```

static files upload under your s3 bucket's static directory

You must recieved no error

<p>7. Install Python3 DJango2 dependencies</p>

```
sudo -H pip3 install -r requirements.txt
```

<p>8. start server</p>

- Local Test
```	
python3 manage.py runserver 8080
```	
or
```	
python3 manage.py runsslserver 8080
```

- Remote Server Test
```
./start_was.sh
```

if you get some error, please check your configurations and AWS 


Start Server
============

All pre-requirements and configurations are complete.
Add crontab your server and was

```
crontab -e
```

add lines below

```
@reboot sudo service nginx start
@reboot sudo systemctl uwsgi start
```

and save, exit


start nginx server and uwsgi was

```
sudo service nginx restart or start
sudo service uwsgi restart or start
```

check uwsgi is no error

```
tail --lines=1000 -f /var/log/syslog
```


