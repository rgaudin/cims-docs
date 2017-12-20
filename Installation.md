# Manual RapidPro Installation

This document describes a step-by-step manual installation of RapidPro. Written on March 2017.

Commands and paths for:

* `debian 8.7`
* `ubuntu 16.04.4` (reference)
* `SLES 12.1 SP1` (CIMS System). As not supported, assuming manual installation of packages.

Assumptions:

* IP is `192.168.60.2`. Change to your hostname.
* All services on the same host (PostgreSQL, redis, mage, nginx, uwsgi, celery)
* PIDs created on `/var/run/rapidpro`
* Logs stored on `/var/log/rapidpro/`
* All services are started by systemd.
* RapidPro is installed under unprivileged user `rapidpro`

## PostgreSQL (as `root`)

``` sh
# debian
apt install postgresql-9.4 postgresql-9.4-postgis-2.1
# ubuntu
apt install postgresql-9.5 postgresql-9.5-postgis-2.2
```

``` sh
sudo su - postgres
psql -c "CREATE USER rapidpro_user WITH PASSWORD 'rapidpro_pass';"
psql -c "CREATE DATABASE rapidpro_db;"
psql -c "GRANT CREATE,CONNECT,TEMP ON DATABASE rapidpro_db to rapidpro_user;"
psql -d rapidpro_db -c 'CREATE EXTENSION postgis; CREATE EXTENSION postgis_topology; CREATE EXTENSION hstore; CREATE EXTENSION "uuid-ossp";'
```

## redis (as `root`)

``` sh
# debian, ubuntu
apt install redis-server redis-tools
```

## SSL Certificate (as `root`)

* Generation of a self-signed SSL certificate for testing. Not suitable for production. Use certbot instead.

``` sh
openssl req -x509 -nodes -days 1095 -newkey rsa:2048 -keyout /etc/ssl/private/rapidpro-self.key -out /etc/ssl/certs/rapidpro-self.crt
```
* Answer as wanted.

## nginx (part 1 – as `root`)

``` sh
# debian, ubuntu
apt install nginx
```

``` sh
mkdir -p /var/log/rapidpro/
```

Edit `/etc/nginx/sites-available/rapidpro`

``` nginx
server {
    listen 80;
    listen [::]:80;

    server_name 192.168.60.2 localhost;

    access_log    /var/log/rapidpro/nginx-access.log combined;
    error_log     /var/log/rapidpro/nginx-error.log;

    # entity size
    client_max_body_size 50m;

    # static files
    location /sitestatic  {
        # alias /home/rapidpro/sitestatic;
    }

    location /media  {
        # alias /home/rapidpro/app/media;
    }

    location / {
        root /tmp;
        autoindex on;
        # proxy_pass        http://localhost:3030;
        proxy_read_timeout 300s;

        proxy_redirect     off;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;

        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    }
}

server {
    listen 443;
    listen [::]:443;

    server_name 192.168.60.2 localhost;

    access_log    /var/log/rapidpro/nginx-ssl-access.log combined;
    error_log     /var/log/rapidpro/nginx-ssl-error.log;

    # certs
    ssl_certificate      /etc/ssl/certs/rapidpro-self.crt;
    ssl_certificate_key  /etc/ssl/private/rapidpro-self.key;

    # ssl opts
    ssl on;
    ssl_protocols        TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers          RC4:HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    keepalive_timeout    70;
    ssl_session_cache    shared:SSL:10m;
    ssl_session_timeout  10m;
    
    # tell client/browser to always use https
    add_header Strict-Transport-Security max-age=31536000;

    # entity size
    client_max_body_size 50m;

    # static files
    location /sitestatic  {
        # alias /home/rapidpro/sitestatic;
    }

    location /media  {
        # alias /home/rapidpro/app/media;
    }

    location / {
        root /tmp;
        autoindex on;
        # proxy_pass        http://localhost:3030;
        proxy_read_timeout 300s;

        proxy_redirect     off;
        proxy_set_header   Host $host;
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Host $server_name;

        # timeouts on unavailable backend(s)
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    }
}
```

``` sh
ln -s /etc/nginx/sites-available/rapidpro /etc/nginx/sites-enabled/rapidpro
nginx -s reload
```

* Test http://192.168.60.2 and https://192.168.60.2. You should see content of `/tmp`.

## temba (as `root`)
``` sh
# debian, ubuntu
apt install git zlib1g liblzma-dev
apt install libncurses5 libncurses-dev libreadline-dev libreadline5
apt install libxslt1.1 xsltproc libxml2-dev libxslt1-dev
apt install libgpg-error-dev libffi6 libffi-dev
apt install perl libpcre3 tcl shadowsocks
apt install libgd3 libjpeg62 libjpeg-dev libpng12-0 libpng-dev
apt install python2.7-dev python-virtualenv python-pip python-lxml libpq-dev python-psycopg2 python-celery
apt install npm nodejs
```

``` sh
npm install -g less coffee-script bower
ln -s /usr/bin/nodejs /usr/bin/node
# create user
useradd -g www-data -m -N -s /bin/bash rapidpro
```

## temba as `rapidpro` user

`su - rapidpro`

``` sh
mkdir ~/venvs && virtualenv ~/venvs/rapidpro
mkdir -p ~/src
git clone https://github.com/rapidpro/rapidpro.git ~/src/rapidpro-`date +"%s"` --depth 1
ln -s ~/src/rapidpro-* ~/app
```
* Add virtualenv activation and jump to temba folder on `~/.bashrc`

``` sh
source ~/venvs/rapidpro/bin/activate
cd ~/app
```

``` sh
pip install -r pip-freeze.txt --allow-all-external
pip install -U pip
pip install -U setuptools
pip install -U uwsgi
```

* Create file `~/settings.py`
**Note**: It's a python file, not a text configuration one. Use `True` and `False` capitalized.

``` python
from __future__ import absolute_import, unicode_literals

from .settings_common import *  # noqa

DEBUG = False
IS_PROD = True
HOSTNAME = '192.168.60.2'
ALLOWED_HOSTS = [HOSTNAME, 'localhost', '127.0.0.1', 'rapidpro']

STATIC_ROOT = "/home/rapidpro/sitestatic"
COMPRESS_ROOT = STATIC_ROOT
MEDIA_ROOT = "/home/rapidpro/app/media"  # must be *inside* temba

USER_TIME_ZONE = 'Africa/Tunis'
LANGUAGE_CODE = 'fr-fr'
DEFAULT_LANGUAGE = "fr-fr"
DEFAULT_SMS_LANGUAGE = "fr-fr"

SECRET_KEY = 'some custom random string here'

POSTGRES_HOST = "localhost"
POSTGRES_USERNAME = "rapidpro_user"
POSTGRES_PASSWORD = "rapidpro_pass"
POSTGRES_NAME = "rapidpro_db"

REDIS_HOST = 'localhost'
REDIS_PORT = 6379
REDIS_DB = 10 if TESTING else 15

IP_ADDRESSES = ('192.168.60.2')  # public IP addresses
ADMINS = (('RapidPro Admin', 'you@email.address'), )
EMAIL_HOST_USER = None
DEFAULT_FROM_EMAIL = None
EMAIL_HOST_PASSWORD = None
EMAIL_USE_TLS = True
EMAIL_HOST = None

TWITTER_API_KEY = '-'
TWITTER_API_SECRET = '-'
MAGE_HOST = "localhost"
MAGE_PORT = 8026
MAGE_AUTH_TOKEN = '123456789abcdef'

# INTERNAL_IPS = INTERNAL_IPS + ('xxx.xx.xx.xx')

# Attention: triggers actual messages and such
SEND_MESSAGES = True
SEND_WEBHOOKS = True
SEND_EMAILS = True

# end of main configuration
TESTING = False
TEMBA_HOST = HOSTNAME
COMPRESS_ROOT = STATIC_ROOT
TEMPLATE_DEBUG = DEBUG
MANAGERS = ADMINS
MAGE_API_URL = "http://{host}:{port}/api/v1".format(host=MAGE_HOST,
                                                    port=MAGE_PORT)


BROKER_URL = 'redis://%s:%d/%d' % (REDIS_HOST, REDIS_PORT, REDIS_DB)
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://%s:%s/%s" % (REDIS_HOST, REDIS_PORT, REDIS_DB),
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}

DATABASES = {
    'default': {
        'ENGINE': 'django.contrib.gis.db.backends.postgis',
        'NAME': POSTGRES_NAME,
        'USER': POSTGRES_USERNAME,
        'PASSWORD': POSTGRES_PASSWORD,
        'HOST': POSTGRES_HOST,
        'PORT': '',
        'ATOMIC_REQUESTS': True,
        'CONN_MAX_AGE': 60,
        'OPTIONS': {
        }
    }
}
DATABASES['default']['CONN_MAX_AGE'] = 60
DATABASES['default']['ATOMIC_REQUESTS'] = True
DATABASES['default']['ENGINE'] = 'django.contrib.gis.db.backends.postgis'

if TESTING:
    INSTALLED_APPS = INSTALLED_APPS + ('storages')
    MIDDLEWARE_CLASSES = ('temba.middleware.ExceptionMiddleware',) \
        + MIDDLEWARE_CLASSES
    CELERY_ALWAYS_EAGER = True
    CELERY_EAGER_PROPAGATES_EXCEPTIONS = True
    BROKER_BACKEND = 'memory'
else:
    CELERY_ALWAYS_EAGER = False
    CELERY_EAGER_PROPAGATES_EXCEPTIONS = True
    BROKER_BACKEND = 'redis'

DEFAULT_DOMAIN = ALLOWED_HOSTS[-1]
BRANDING_URL = "https://{}".format(DEFAULT_DOMAIN)
BRANDING = {
    'default': {
        'slug': 'rapidpro',
        'name': 'RapidPro',
        'org': 'UNICEF',
        'styles': ['brands/rapidpro/font/style.css',
                   'brands/rapidpro/less/style.less'],
        'welcome_topup': 1000,
        'email': 'join@you.domain',
        'support_email': 'support@rapidpro.ona.io',
        'link': BRANDING_URL,
        'api_link': BRANDING_URL,
        'docs_link': BRANDING_URL,
        'domain': DEFAULT_DOMAIN,
        'favico': 'brands/rapidpro/rapidpro.ico',
        'splash': '/brands/rapidpro/splash.jpg',
        'logo': '/brands/rapidpro/logo.png',
        'allow_signups': True,
        'tiers': dict(multi_user=0, multi_org=0),
        'bundles': [],
        'welcome_packs': [dict(size=5000, name="Demo Account"),
                          dict(size=100000, name="UNICEF Account")],
        'description': "My Custom RapidPro",
        'credits': "Copyright &copy; 2012-2015 UNICEF, Nyaruka. "
                   "All Rights Reserved.",
    }
}
DEFAULT_BRAND = 'default'
```

``` sh
ln -sf ~/settings.py ~/app/temba/settings.py
mkdir -p ~/{sitestatic,media}
chmod 755 ~/{sitestatic,media}
cd ~/app
./manage.py migrate --noinput
./manage.py collectstatic --noinput
./manage.py createsuperuser --username root
```
Remember the password you set for root (ex: admin)

* Create file `~/app/uwsgi.ini`

``` ini
[uwsgi]
http=:3030
# socket=/var/run/rapidpro/rapidpro.sock
# chmod-socket=777

uid=rapidpro
gid=www-data
chdir=/home/rapidpro/app
pidfile=/var/run/rapidpro/uwsgi.pid
logto=/var/log/rapidpro/uwsgi_rapidpro.log

virtualenv=/home/rapidpro/venvs/rapidpro
module=temba.wsgi:application
env=HTTPS=on

master=True

# static-map=/sitestatic=/home/rapidpro/sitestatic # if no frontend httpd

# stats=/var/run/rapidpro/uwsgi_stats.sock

# tweak for perf
processes = 4
threads = 2  # nb cpu
enable-threads = True
disable-logging = True

buffer-size=8192
harakiri=240                # respawn processes taking more than 240 seconds
max-requests=5000           # respawn processes after serving 5000 requests
vacuum=True                 # clear environment on exit
```

``` sh
./manage.py runserver 0.0.0.0:8000
```
* Test setup on http://192.168.60.2:8000

## temba (as `root`)

* Create file `/etc/systemd/system/rapidpro.service`

``` ini
[Unit]
Description=uWSGI instance to serve rapidpro

[Service]
Type=notify
StandardError=syslog
NotifyAccess=all

PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /var/run/rapidpro/
ExecStartPre=/bin/chown -R rapidpro:www-data /var/run/rapidpro/

ExecStart=/home/rapidpro/venvs/rapidpro/bin/uwsgi --ini /home/rapidpro/app/uwsgi.ini --env DJANGO_SETTINGS_MODULE=temba.settings
Restart=always
KillSignal=SIGQUIT

[Install]
WantedBy=multi-user.target
```

``` sh
mkdir -p /var/log/rapidpro/
chgrp www-data /var/log/rapidpro/
chmod 775 /var/log/rapidpro/

systemctl status rapidpro
systemctl daemon-reload
systemctl enable rapidpro
systemctl start rapidpro
```

## nginx (part 2)

* Edit `/etc/nginx/sites-available/rapidpro`:
 * remove comments on `alias` commands.
 * remove comments on `proxy_pass` commands.
 * remove lines starting with `root` and `autoindex`

``` sh
nginx -s reload
```

* Test http://192.168.60.2 and https://192.168.60.2

## GIS data (as `rapidpro` user)

`su - rapidpro`

**Note**: Following example is for Tunisia aka `R192757`. Replace with your country's ID from [nominatim](https://nominatim.openstreetmap.org/). More details on [RapidPro Hosting](https://rapidpro.github.io/rapidpro/docs/hosting/).

``` sh
mkdir -p ~/src/posm
# Tunisia example
wget https://github.com/nyaruka/posm-extracts/raw/master/geojson/R192757admin0_simplified.json -O ~/src/posm/R192757admin0_simplified.json
wget https://github.com/nyaruka/posm-extracts/raw/master/geojson/R192757admin1_simplified.json -O ~/src/posm/R192757admin1_simplified.json
wget https://github.com/nyaruka/posm-extracts/raw/master/geojson/R192757admin2_simplified.json -O ~/src/posm/R192757admin2_simplified.json
cd ~/app
./manage.py import_geojson ~/src/posm/*_simplified.json
```

## Web UI Configuration

* Log into https://192.168.60.2/org/grant/ with root credentials
* Create Organization Owner account
* Go to https://192.168.60.2/org/home/ and change Location appropriately.
* Log out and login with newly created account.

## Celery (as `root`)

* Create file `/etc/systemd/system/celerybeat.service`

``` ini
[Unit]
Description=Celery beat for rapidpro
After=network.target

[Service]
User=rapidpro
Group=www-data
WorkingDirectory=/home/rapidpro/app

PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /var/run/rapidpro/
ExecStartPre=/bin/chown -R rapidpro:www-data /var/run/rapidpro/

ExecStart=/home/rapidpro/venvs/rapidpro/bin/celery beat -A temba --logfile=/var/log/rapidpro/celerybeat.log --pidfile=/var/run/rapidpro/celerybeat.pid

Restart=always

[Install]
WantedBy=multi-user.target
```
``` sh
systemctl status celerybeat
systemctl daemon-reload
systemctl enable celerybeat
systemctl start celerybeat
systemctl status celerybeat
```
* Create file `/etc/systemd/system/celery.service`

``` ini
[Unit]
Description=Celery Service
After=network.target

[Service]
Type=forking
User=rapidpro
Group=www-data
WorkingDirectory=/home/rapidpro/app

PermissionsStartOnly=true
ExecStartPre=/bin/mkdir -p /var/run/rapidpro/
ExecStartPre=/bin/chown -R rapidpro:www-data /var/run/rapidpro/

ExecStart=/home/rapidpro/venvs/rapidpro/bin/celery multi start default-node handler-node msg-node flow-node -A temba --logfile="/var/log/rapidpro/celery-%%n%%I.log" --pidfile="/var/run/rapidpro/celery-%%n.pid" --time-limit=300 --concurrency=1 -Q:default-node celery -Q:handler-node handler -Q:msg-node msgs -Q:flow-node flows
ExecStop=/home/rapidpro/venvs/rapidpro/bin/celery multi stopwait default-node handler-node msg-node flow-node --pidfile="/var/run/rapidpro/celery-%%n.pid"
ExecReload=/home/rapidpro/venvs/rapidpro/bin/celery multi restart default-node handler-node msg-node flow-node -A temba --pidfile="/var/run/rapidpro/celery-%%n.pid" --logfile="/var/log/rapidpro/celery-%%n%%I.log" --loglevel="INFO" --time-limit=300 --concurrency=1 -Q:default-node celery -Q:handler-node handler -Q:msg-node msgs -Q:flow-node flows

[Install]
WantedBy=multi-user.target
```
``` sh
systemctl status celery
systemctl daemon-reload
systemctl enable celery
systemctl start celery
systemctl status celery
```

## Twitter Channel

### Create Twitter App

* Create Twitter account
* Go to https://dev.twitter.com/apps/
* Create new app
* *Permission*: Read, Write and Access direct messages
* *Settings*: set callback_url `http://rapidpro/channels/channel/claim_twitter/`. *Note*: `callback_url` if not enforced by default but edit it to your public domain if you have one.
* *Keys and Access Tokens*: copy `API Key` and `API Secret`.

**Attention**: if Access Level is not `Read, write, and direct messages`, then regenerate token.

### Configure temba
* Update `~/settings.py`:

```python
TWITTER_API_KEY = 'XXX'
TWITTER_API_SECRET = 'XXX'
```

### Restart temba and stop celery (as `root`)
``` sh
systemctl restart rapidpro
systemctl stop celery
systemctl stop celerybeat
```
* Create Twitter channel at http://192.168.60.2/channels/channel/claim_twitter/

### Install mage (as `root`)

``` sh
# ubuntu
apt install software-properties-common
add-apt-repository "deb http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main"
apt update
apt install oracle-java8-installer
update-alternatives --config java
```

### Configure mage (as `rapidpro` user)

`su - rapidpro`

``` sh
wget -O ~/src/mage-0.1.82.jar "http://s000.tinyupload.com/download.php?file_id=57690352709769539444&t=5769035270976953944441119"`
mkdir -p ~/mage`
ln -s ~/src/mage-*.jar ~/mage/mage.jar
```

* Get API Token from https://192.168.60.2/api/v2
* Create file `~/mage/config.yml`

``` yaml
#
# Configuration for Message Mage
#
# The following environment variables should be defined:
#  * PRODUCTION - whether this is a production instance (1) or not (0)
#  * DATABASE_URL - Django format database connection URL
#  * REDIS_HOST - Redis cache host
#  * REDIS_DATABASE - Redis database number
#  * TEMBA_HOST - Temba host name
#  * TEMBA_AUTH_TOKEN - authentication token for the Temba API
#  * TWITTER_API_KEY - public key for Twitter API
#  * TWITTER_API_SECRET - secret token for Twitter API
#  * SEGMENTIO_WRITE_KEY - write key for segment.io
#  * SENTRY_DSN - URL for Sentry error reporting
#  * LIBRATO_EMAIL - Librato email/login
#  * LIBRATO_API_TOKEN - Librato API token
#

general:
    production: ${PRODUCTION}

server:
    type: default
    minThreads: 8
    maxThreads: 64
    applicationContextPath: /
    applicationConnectors:
      - type: http
        port: 8026
    rootPath: '/api/v1/*'
    adminContextPath: /
    adminConnectors:
      - type: http
        port: 8028

database:
    # the name of your JDBC driver
    driverClass: org.postgresql.Driver

    # the URL (same format as Django)
    fullUrl: ${DATABASE_URL}

    # any properties specific to your JDBC driver:
    properties:
        charSet: UTF-8

    # the maximum amount of time to wait on an empty pool before throwing an exception
    maxWaitForConnection: 1s

    # the SQL query to run when validating a connection's liveness
    validationQuery: "/* MyService Health Check */ SELECT 1"

    # the minimum number of connections to keep open
    minSize: 8

    # the maximum number of connections to keep open
    maxSize: 32

    # whether or not idle connections should be validated
    checkConnectionWhileIdle: false

logging:
    appenders:
      - type: console
        threshold: INFO
        target: stdout
    loggers:
        "com.sun.jersey.api.container.filter.LoggingFilter": DEBUG

redis:
    host: ${REDIS_HOST}
    database: ${REDIS_DATABASE}

temba:
    apiUrl: https://${TEMBA_HOST}/api/v1
    authToken: ${TEMBA_AUTH_TOKEN}

twitter:
    apiKey: ${TWITTER_API_KEY}
    apiSecret: ${TWITTER_API_SECRET}

monitoring:
    segmentioWriteKey: ${SEGMENTIO_WRITE_KEY}
    sentryDsn: ${SENTRY_DSN}
    libratoEmail: ${LIBRATO_EMAIL}
    libratoApiToken: ${LIBRATO_API_TOKEN}
```
* Create and **configure** file `~/mage/environ.sh`

``` sh
PRODUCTION=1
DATABASE_URL="postgresql://rapidpro_user:rapidpro_pass@localhost/rapidpro_db"
REDIS_HOST=localhost
REDIS_DATABASE=8
TEMBA_HOST=192.168.60.2
TEMBA_AUTH_TOKEN="xxxx"
TWITTER_API_KEY="xxxx"
TWITTER_API_SECRET="xxxx"
TEMBA_NO_SSL=1
SEGMENTIO_WRITE_KEY="false"
SENTRY_DSN="false"
LIBRATO_EMAIL="false"
LIBRATO_API_TOKEN="false"
```

* `set -a && source ~/mage/environ.sh && set +a && java -jar ~/mage/mage.jar server ~/mage/config.yml`
* Check mage output to find “*Found 1 active Twitter channel(s)*” et “*Added Twitter stream for handle*”

### Create mage startup (as `root`)
* Create file `vim /etc/systemd/system/mage.service`

``` ini
[Unit]
Description=mage Twitter service

[Service]
Type=simple
User=rapidpro
Group=www-data
WorkingDirectory=/home/rapidpro/mage
EnvironmentFile=/home/rapidpro/mage/environ.sh

ExecStart=/bin/sh -c "exec java -jar /home/rapidpro/mage/mage.jar server /home/rapidpro/mage/config.yml"

Restart=always

[Install]
WantedBy=multi-user.target
```

``` sh
systemctl status mage
systemctl daemon-reload
systemctl enable mage
systemctl start mage
systemctl start celery
systemctl start celerybeat
```

## Facebook Channel

Facebook requires your callback to be on a genuine SSL certificate (actual domain, not self-signed, major authority).

### Genuine SSL Certificate (on a private network)

* Create a tunnel from host computer to gateway public server targeting VM's 443
* `ssh user@gatewayserver -R localhost:8000:192.168.60.2:80`
* Configure nginx for public domain name

``` nginx
    server_name 192.168.60.2 pubdomain.com;
    
    ssl_certificate      /etc/ssl/certs/pubdomain.com/fullchain.pem;
    ssl_certificate_key  /etc/ssl/certs/pubdomain.com/privkey.pem;
```
* Test it on https://pubdomain.com:8000/

### Facebook
* Create an Application at https://developer.facebook.com/ (type *Messenger App*)
* Go to section *Messenger*
* Select your page and generate Token
* Subscribe your page to *Messenger*
* Add `pages_messaging` and `pages_messaging_subscriptions`. `pages_messaging_phone_number` is optionnal.

### RapidPro
* Create Channel from https://pubdomain.com:8000/channels/channel/claim_facebook/
* Copy the token from your application
* Add Webhook and indicate callback (tick everything)
* Edit the Facebook Page settings to enable messenger: “Instant answer” settings indicate it's a bot.
* Test using one of the Page's admin account.
* Submit your app to Facebook from Application page (tick `pages_messaging` and `pages_messaging_subscriptions`)


# Updating the codebase

RapidPro is an active project with continuous updates: new features, bug and security fixes. It is recommended to update frequently.

## Take it down

``` sh
systemctl stop rapidpro
systemctl stop celery
systemctl stop celerybeat
systemctl stop mage
systemctl stop redis-server
```

## Backup database

``` sh
su - postgres -c "pg_dump -C rapidpro > /tmp/rapidpro-dump.sql"
```

## Get the latest code (as `rapidpro` user)

**Note**: Change `rapidpro-XXX` with the latest version.

``` sh
git clone https://github.com/rapidpro/rapidpro.git ~/src/rapidpro-`date +"%s"` --depth 1
ln -sf ~/src/rapidpro-XXX ~/app
ln -sf ~/settings.py ~/app/temba/settings.py
pip install -r pip-freeze.txt --allow-all-external
pip install -U pip
pip install -U setuptools
pip install -U uwsgi
./manage.py migrate --noinput
./manage.py collectstatic --noinput
```

## Restart (as `root`)

``` sh
systemctl start redis-server
systemctl start celery
systemctl start celerybeat
systemctl start rapidpro
systemctl start mage
```

* Test the system, include async tasks like messages and channels.

## Rollback (if it's not working)

Potential problems:

* Unable to install new pip requirements.
* Pip requirements brought new dependencies that can't be installed.
* Migrations failed to apply.
* Missing/changed settings prevent RapidPro from working correctly.

Check [developers list](https://groups.google.com/forum/#!forum/rapidpro-dev) to find out and get advices.

``` sh
systemctl stop rapidpro
systemctl stop celery
systemctl stop celerybeat
systemctl stop mage
systemctl stop redis-server
```

### Restore database dump

Only if you applied at least one migration (even if it failed) during the update. Skip if there was no new migration or if it failed before that step.

``` sh
su - postgres -c "psql -c 'DROP DATABASE rapidpro_db;'"
su - postgres -c "psql --set ON_ERROR_STOP=on < /tmp/rapidpro-dump.sql"
```

### Rollback code (as `rapidpro` user)

**Note**: Change `rapidpro-XXX` to the latest working version. Use `ls -lt ~/src/` to find it.

``` sh
ln -sf ~/src/rapidpro-XXX ~/app
ln -sf ~/settings.py ~/app/temba/settings.py
pip install -r pip-freeze.txt --allow-all-external
pip install -U pip
pip install -U setuptools
pip install -U uwsgi
./manage.py migrate --noinput
./manage.py collectstatic --noinput
```

### Restart (as `root`)

``` sh
systemctl start redis-server
systemctl start celery
systemctl start celerybeat
systemctl start rapidpro
systemctl start

