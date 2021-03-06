Deployment
==================================

.. note::
    This tutorial is only tested on Debian Buster!

Firewall
--------

Ports ``80`` & ``443`` need to be open, for installing the dependencies & running
the server. An example for iptables to open the ports.::

    $ iptables -A INPUT -p tcp --dport 443 -j ACCEPT
    $ iptables -A OUTPUT -p tcp --dport 443 -j ACCEPT
    $ iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    $ iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT

Dependent Services
------------------

Postgres 12 with Postgis 3
..........................

To use the performance boost of the new Postgres Version, add the
`postgres repo <https://www.postgresql.org/download/linux/debian/>`_
from postgresql.org to your system.::

    $ apt-get --no-install-recommends install \
        locales gnupg2 wget ca-certificates rpl pwgen software-properties-common gdal-bin iputils-ping
    $ sh -c "echo \"deb http://apt.postgresql.org/pub/repos/apt/ buster-pgdg main\" > /etc/apt/sources.list.d/pgdg.list"
    $ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
    $ apt-get update

Install Postgres 12 with Postgis 3.::

    $ apt-get --no-install-recommends install postgresql-client-12 \
        postgresql-common postgresql-12 postgresql-12-postgis-3 \
        netcat postgresql-12-ogr-fdw postgresql-12-postgis-3-scripts \
        postgresql-12-cron postgresql-plpython3-12 postgresql-12-pgrouting

Postgres's service is started & set to come up after each system reboot.::

    $ systemctl status postgresql.service

While the installation, the user ``postgres`` was added to the system. With the
user you can access the admin postgres user.

    $ su - postgres

Change the postgres user ``postgres`` password (remember this password!)::

    $ psql -c "alter user postgres with password 'YourNewPassword'"

Now access the postgres prompt.::

    $ psql

Enable Postgis & hstore extensions for postgres.::

    $ CREATE EXTENSION postgis;
    $ CREATE EXTENSION hstore;
    $ CREATE EXTENSION postgis_topology;

Set the template postgres database to ``UTF-8`` encoding.::

    $ UPDATE pg_database SET datistemplate = FALSE WHERE datname = 'template1';
    $ DROP DATABASE template1;
    $ CREATE DATABASE template1 WITH TEMPLATE = template0 ENCODING = 'UTF8';
    $ UPDATE pg_database SET datistemplate = TRUE WHERE datname = 'template1';
    $ \c template1
    $ VACUUM FREEZE;

Create the ``gis`` database with the user ``mapnik`` to access the ``gis`` database.::

    $ CREATE DATABASE gis;
    $ CREATE USER mapnik WITH ENCRYPTED PASSWORD 'MyStr0ngP@SS';
    $ GRANT ALL PRIVILEGES ON DATABASE gis to mapnik;

Set the new ``mapnik`` database user as superuser.::

    $ ALTER USER mapnik WITH SUPERUSER;

Logout from postgres prompt & user.::

    $ \q
    $ exit

Redis 5
.......

Redis use as a caching server for the tiles & as a task broker for celery.

For installing redis server, use::

    $ apt-get install --no-install-recommends redis-server

If you running redis on the same system as the web-service, then is redis ready
to work :)

NGINX
......

Install NGINX and certbot for Let's Encrypt.::

    $ apt-get install --no-install-recommends nginx python3-acme \
        python3-certbot python3-mock python3-openssl python3-pkg-resources \
        python3-pyparsing python3-zope.interface python3-certbot-nginx

Create a NGINX config file for ohdm.::

    $ nano /etc/nginx/sites-available/MapnikTileServer.conf

    server {
        server_name a.ohdm.net b.ohdm.net c.ohdm.net;

        location = /favicon.ico { access_log off; log_not_found off; }
        location  /static/ {
        alias /home/mapnik/MapnikTileServer/staticfiles/;
        }

        location / {
            include proxy_params;
            proxy_read_timeout 360;
            proxy_pass http://unix:/home/mapnik/MapnikTileServer/MapnikTileServer.sock;
        }

    }

    server {
        server_name monitor.ohdm.net;

        location / {
            include proxy_params;
            proxy_pass http://127.0.0.1:5555;
        }
    }

.. note::
    Change the domains ``a.ohdm.net``, ``b.ohdm.net``, ``c.ohdm.net`` & ``monitor.ohdm.net``
    in the NGINX config file to your domains!

Link the config file from ``/etc/nginx/sites-available/MapnikTileServer.conf``
to ``/etc/nginx/sites-enabled/MapnikTileServer.conf``.::

    $ ln -s /etc/nginx/sites-available/MapnikTileServer.conf /etc/nginx/sites-enabled

Test if the config was set up right & restart NGINX.::

    $ nginx -t
    $ systemctl restart nginx

Obtaining an SSL Certificate.::

    $ certbot --nginx -d a.ohdm.net -d b.ohdm.net -d c.ohdm.net -d monitor.ohdm.net
    2
    2

Test if Let's Encrypt was sucessfully set up.::

    $ nginx -t
    $ systemctl restart nginx

Test if certbot can auto renew the SSL certificate.::

    $ certbot renew --dry-run

Install MapnikTileServer
------------------------

System dependencies::

    $ apt-get install --no-install-recommends wget unzip fontconfig gnupg

Node::

    $ apt-get install nodejs npm
    $ npm i -g npm@^6

Python::

    $ apt-get install --no-install-recommends python3-pip python3-dev \
        python3-setuptools

Mapnik-utils for openstreetmap-carto::

    $ apt-get install --no-install-recommends mapnik-utils

Dependencies for building Python packages::

    $ apt-get install --no-install-recommends build-essential

Psycopg2 dependencies::

    $ apt-get install --no-install-recommends libpq-dev

Translations dependencies::

    $ apt-get install --no-install-recommends gettext

Fonts for mapnik::

    $ apt-get install --no-install-recommends fonts-dejavu fonts-hanazono \
    ttf-unifont \
    fonts-noto fonts-noto-cjk fonts-noto-cjk-extra fonts-noto-color-emoji \
    fonts-noto-hinted fonts-noto-mono \
    fonts-noto-unhinted \
    fonts-noto-extra fonts-noto-ui-core fonts-noto-ui-extra

`Geodjango <https://docs.djangoproject.com/en/3.0/ref/contrib/gis/install/geolibs/>`_::

    $ apt-get install --no-install-recommends binutils libproj-dev gdal-bin

Git::

    $ apt-get install --no-install-recommends git

Mapnik::

    $ apt-get install --no-install-recommends libmapnik-dev libmapnik3.0 mapnik-utils \
    python3-mapnik

Supervisor::

    $ apt-get install --no-install-recommends supervisor

Download & install more `noto fonts <https://www.google.com/get/noto/>`_ for mapnik::

    $ mkdir noto-fonts
    $ cd noto-fonts
    $ wget https://noto-website-2.storage.googleapis.com/pkgs/NotoSansBalinese-unhinted.zip
    $ wget https://noto-website-2.storage.googleapis.com/pkgs/NotoSansSyriacEastern-unhinted.zip
    $ wget https://noto-website-2.storage.googleapis.com/pkgs/NotoColorEmoji-unhinted.zip
    $ wget https://noto-website-2.storage.googleapis.com/pkgs/NotoEmoji-unhinted.zip
    $ unzip -o \*.zip
    $ cp ./*.ttf /usr/share/fonts/truetype/noto/
    $ fc-cache -fv
    $ fc-list
    $ cd ..
    $ rm -r noto-fonts

Update NodeJS to the latest stable::

    $ npm install -g n stable

Install `CartoCSS <https://github.com/mapbox/carto>`_ with a version below 1.::

    $ npm install -g carto@0

Set environment vars for running the MapnikTileServer.::

    $ nano /etc/environment

Fill the ``/etc/environment`` file with the following values.

    # Django
    DJANGO_READ_DOT_ENV_FILE=True
    DJANGO_SETTINGS_MODULE=config.settings.production

Create a Mapnik user, for running the MapnikTileServer.::

    $ adduser mapnik

Log into ``mapnik`` user and go to the home folder.::

    $ su - mapnik
    $ cd

Download `openstreetmap-carto <https://github.com/linuxluigi/openstreetmap-carto/>`_::

    $ git clone https://github.com/linuxluigi/openstreetmap-carto.git

Go to the new openstreetmap-carto folder, download the shape files & create
the default mapnik style XML::

    $ cd openstreetmap-carto
    $ ./scripts/get-shapefiles.py
    $ carto project.mml > style.xml

Next go back to the ``mapnik`` home folder.::

    $ cd

Download `MapnikTileServer <https://github.com/OpenHistoricalDataMap/MapnikTileServer/>`_.::

    $ git clone https://github.com/OpenHistoricalDataMap/MapnikTileServer.git
    $ cd MapnikTileServer

Install / update the python packages as root user.::

    $ exit
    $ pip3 install -r /home/mapnik/MapnikTileServer/requirements/system.txt
    $ pip3 install -r /home/mapnik/MapnikTileServer/requirements/base.txt
    $ pip3 install -r /home/mapnik/MapnikTileServer/requirements/production.txt

.. note::
    When install an update of MapnikTileServer, also update the python packages!

Go back to the ``mapnik`` user & back to the MapnikTileServer folder.::

    $ su mapnik
    $ cd /home/mapnik/MapnikTileServer

Create a ``.env`` file for the MapnikTileServer settings. Go to :ref:`settings`
to see all possibles options. Below is a minimal configuration::

    # General
    # ------------------------------------------------------------------------------
    DJANGO_SECRET_KEY=!!!ChangeMeToSomeRandomValue!!!!!
    DJANGO_ALLOWED_HOSTS=a.ohdm.net,b.ohdm.net,c.ohdm.net

    # Redis
    # ------------------------------------------------------------------------------
    REDIS_URL=redis://localhost:6379/0
    CELERY_BROKER_URL=redis://localhost:6379/0

    # ohdm
    # ------------------------------------------------------------------------------
    CARTO_STYLE_PATH=/home/mapnik/openstreetmap-carto

    # Default PostgreSQL
    # ------------------------------------------------------------------------------
    DATABASE_URL="postgres://mapnik:MyStr0ngP@SS@localhost:5432/gis"
    POSTGRES_HOST=localhost
    POSTGRES_PORT=5432
    POSTGRES_DB=gis
    POSTGRES_USER=mapnik
    POSTGRES_PASSWORD=MyStr0ngP
    PGCONNECT_TIMEOUT=60

    # OHDM PostgreSQL
    # ------------------------------------------------------------------------------
    OHDM_SCHEMA=ohdm

Tests the settings, migrate the database, set indexes & collect static files::

    $ python3 manage.py migrate
    $ python3 manage.py set_indexes
    $ python3 manage.py collectstatic

Add a superuser for the admin panel.::

    $ python3 manage.py createsuperuser

Add ``supervisor`` script to auto start django, celery & flower at system start.
For creating the scripts, go back to the root user.::

    $ exit

Open the text editor to create the ``supervisor`` file.::

    $ nano /etc/supervisor/conf.d/mapnik_tile_server.conf

Fill the ``supervisor`` file with the values below, but don't forget to change
``--basic_auth="ChangeMeFlowerUser:ChangeMeFlowerPassword"`` with your flower user & password.::

    [supervisord]
    environment=DJANGO_READ_DOT_ENV_FILE=True,DJANGO_SETTINGS_MODULE=config.settings.production,CELERY_BROKER_URL=redis://localhost:6379/0

    [program:MapnikTileServer_celery_worker]
    command=celery -A config.celery_app worker -l INFO
    user=mapnik
    directory=/home/mapnik/MapnikTileServer
    autostart=true
    autorestart=true
    priority=10
    stderr_logfile=/var/log/MapnikTileServer_celery_worker.err.log

    [program:MapnikTileServer_celery_beat]
    command=celery -A config.celery_app beat -l INFO
    user=mapnik
    directory=/home/mapnik/MapnikTileServer
    autostart=true
    autorestart=true
    priority=10
    stderr_logfile=/var/log/MapnikTileServer_celery_beat.err.log

    [program:MapnikTileServer_celery_flower]
    command=celery flower --app=config.celery_app --broker="redis://localhost:6379/0" --basic_auth="ChangeMeFlowerUser:ChangeMeFlowerPassword"
    user=mapnik
    directory=/home/mapnik/MapnikTileServer
    autostart=true
    autorestart=true
    priority=10
    stderr_logfile=/var/log/MapnikTileServer_celery_flower.err.log

    [program:MapnikTileServer_django]
    command=/usr/local/bin/gunicorn config.wsgi --workers 2 --bind unix:/home/mapnik/MapnikTileServer/MapnikTileServer.sock -t 360
    user=mapnik
    directory=/home/mapnik/MapnikTileServer
    autostart=true
    autorestart=true
    priority=10
    stderr_logfile=/var/log/MapnikTileServer_django.err.log

To enable the ``supervisor`` script.::

    $ supervisorctl reread
    $ supervisorctl update
    $ supervisorctl start all
    $ supervisorctl status

Use commands
------------

For using django commands from :ref:`commands`, log into the ``mapnik`` user &
go to the ``/home/mapnik/MapnikTileServer``.::

    $ su mapnik
    $ cd /home/mapnik/MapnikTileServer

The commands in :ref:`commands` are written for the docker usage, to use them
without docker, just use the command after the ``django`` keyword. For example,
to use ``set_indexes``, in the docs the command is written down as
``docker-compose -f local.yml run --rm django python manage.py set_indexes`` and
to use it without docker, just use ``python3 manage.py set_indexes``.

Download updates
----------------

Stop all services first.::

    $ supervisorctl stop all

Log into the ``mapnik`` user and go to the openstreetmap-carto folder.::

    $ su mapnik
    $ cd /home/mapnik/openstreetmap-carto

Get the latest version with ``git pull``.::

    $ git pull

Download the latest shape files & create the default mapnik style XML.::

    $ ./scripts/get-shapefiles.py
    $ carto project.mml > style.xml

Go to the MapnikTileServer.::

    $ cd /home/mapnik/MapnikTileServer

Download the latest code from github, for the MapnikTileServer.::

    $ git pull

Update the database & static files.::

    $ python3 manage.py migrate
    $ python3 manage.py set_indexes
    $ python3 manage.py collectstatic

Log out from the ``mapnik`` user & start the web services again.::

    $ exit
    $ supervisorctl start all

Remove all packages were automatically installed and are no longer required.::

    $ apt autoremove

On ``monitor.ohdm.net`` you should now able to log into the flower celery monitor.
The user information is in ``/etc/supervisor/conf.d/mapnik_tile_server.conf`` on the
line ``--basic_auth="ChangeMeFlowerUser:ChangeMeFlowerPassword"``.
