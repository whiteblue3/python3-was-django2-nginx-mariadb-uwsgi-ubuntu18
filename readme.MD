Pre-Requirements
================

```
Ubuntu >= 18.04 LTS
Python >= 3.6.6
DJango >= 2.1.2
uWSGI
NginX
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


NginX Configuration
===================

<p>1. install server nginx</p>

```
sudo apt-get install nginx nginx-extras
```

<p>2. configure /etc/nginx/nginx.conf<p>

- set server_tokens off
```
http {
...
	server_tokens off;
...
}
```

- set TLS version
```
http {
...
	ssl_protocols TLSv1.2;
...
}
```

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
        more_set_headers "Server: server"; # Hide and change your server header

        set_real_ip_from    0.0.0.0/0;
        real_ip_header      X-Forwarded-For; # Set access log to real ip
...
}
```

<p>3. configure /etc/nginx/sites-available/default :</p>

- add was upstream
```
upstream api_server {
        server 127.0.0.1:8080 fail_timeout=0;
}
```

- additionally you can redirect http to https
```
server {
        listen 80;
        listen [::]:80;

        return 301 https://$host$request_uri?;
}
```

- additionally you can configure default information
```
server {
...
	listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;
        charset utf-8;

        # SSL configuration
        #
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
        add_header X-Frame-Options "DENY";
        add_header X-Content-Type-Options "nosniff";
        add_header Cache-Control "no-cache";
        add_header X-XSS-Protection "1; mode=block";
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

- allow and enable your python was
```
server {
...
	location / {
                try_files $uri $uri/ =404;

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

<p>3. install uwsgi via pip3 and enable it</p>

```
sudo -H pip3 install uwsgi
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



DJango WAS Project Installations
================================

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

<p>6. test associate with aws s3</p>

```
	python3 manage.py collectstatic
```

You must recieved no error

<p>7. start server</p>

- Local Test
```	
python3 manage.py runserver 8080
```	
or
```	
python3 manage.py runsslserver 8080
```

- Remote Server (Test)
```
./start_was.sh
```

- Remote Server (Live)
```
sudo service uwsgi restart
```


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

and ctrl+x -> y to save and exit


start nginx server and uwsgi was

```
sudo service nginx restart or start
sudo service uwsgi restart or start
```

check uwsgi is no error

```
tail --lines=1000 -f /var/log/syslog
```


