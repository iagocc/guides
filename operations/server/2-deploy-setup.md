# Setting up an app deployment server

## Prerequisites

* [Setup server from scratch](./setup.md)

Connect to the server using ssh:

```
ssh [user]@[ip-address] -p [port]
```

## Change the root password

```
sudo passwd
```

## Install some required packages
```
sudo apt-get install build-essential git-core libreadline-gplv2-dev curl
libssl-dev libxslt-dev libxml2-dev libpcre3-dev libpcrecpp0 unzip
python-software-properties
```
## Install nginx with pagespeed module.

Create the nginx user

```
sudo useradd -m nginx
```

Create a temporary folder for building nginx

```
mkdir ~/sources
cd ~/sources/
```

Download and extract the pagespeed module for nginx

```
wget
https://github.com/pagespeed/ngx_pagespeed/archive/release-1.9.32.1-beta.zip
unzip release-1.9.32.1-beta.zip
```

Download and extract the psol library to the pagespeed module.

```
cd ngx_pagespeed-release-1.9.32.1-beta/
wget https://dl.google.com/dl/page-speed/psol/1.9.32.1.tar.gz
tar -xzvf 1.9.32.1.tar.gz # expands to psol/
```

Download and extract nginx.

```
cd ~/sources
wget http://nginx.org/download/nginx-1.7.6.tar.gz
tar -xvzf nginx-1.7.6.tar.gz
```

Configure nginx with the pagespeed and spdy modules.

```
cd nginx-1.7.6/

./configure \
--user=nginx                          \
--prefix=/etc/nginx                   \
--sbin-path=/usr/sbin/nginx           \
--conf-path=/etc/nginx/nginx.conf     \
--pid-path=/var/run/nginx.pid         \
--lock-path=/var/run/nginx.lock       \
--error-log-path=/var/log/nginx/error.log \
--http-log-path=/var/log/nginx/access.log \
--with-http_gzip_static_module        \
--with-http_stub_status_module        \
--with-http_ssl_module                \
--with-http_spdy_module               \
--with-pcre                           \
--with-file-aio                       \
--with-http_realip_module             \
--add-module=$HOME/sources/ngx_pagespeed-release-1.9.32.1-beta \
--without-http_scgi_module            \
--without-http_uwsgi_module           \
--without-http_fastcgi_module         \
```

Build and install nginx.

```
make
sudo make install
```

Remove the temporary sources directory.

```
rm -rf ~/sources
```

Create the sites-enabled folder so we can add multiple sites.

```
sudo mkdir /etc/nginx/sites-enabled
sudo mkdir /etc/nginx/sites-available
```

Replace /etc/nginx/nginx.conf with the following to support multiple sites and
pagespeed

```
user nginx;
worker_processes  4;

events {
  worker_connections  1024;
}

http {

  # Basic settings

  sendfile on;
  tcp_nopush on;
  tcp_nodelay on;
  keepalive_timeout 65;
  types_hash_bucket_size 64;
  types_hash_max_size 2048;
  server_names_hash_bucket_size 64;

  include mime.types;
  default_type application/octet-stream;

  # Gzip settings

  gzip on;
  gzip_http_version 1.1;
  gzip_vary on;
  gzip_comp_level 6;
  gzip_proxied any;
  gzip_types text/plain text/html text/css application/json
application/javascript application/x-javascript text/javascript text/xml
application/xml application/rss+xml application/atom+xml application/rdf+xml;
  gzip_buffers 128 4k;
  gzip_disable "MSIE [1-6]\.(?!.*SV1)";

  # Pagespeed settings

  pagespeed on;
  pagespeed FileCachePath "/var/cache/ngx_pagespeed/";

  # Logging settings

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  # Virtual hosts

  include /etc/nginx/sites-enabled/*;
}
```

Add the following init script to /etc/init.d/nginx

```
#! /bin/sh

### BEGIN INIT INFO
# Provides:          nginx
# Required-Start:    $all
# Required-Stop:     $all
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts the nginx web server
# Description:       starts nginx using start-stop-daemon
### END INIT INFO

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
DAEMON=/usr/sbin/nginx
NAME=nginx
DESC=nginx

test -x $DAEMON || exit 0

# Include nginx defaults if available
if [ -f /etc/default/nginx ] ; then
    . /etc/default/nginx
fi

set -e

. /lib/lsb/init-functions

case "$1" in
  start)
    echo -n "Starting $DESC: "
    start-stop-daemon --start --quiet --pidfile /var/run/$NAME.pid \
        --exec $DAEMON -- $DAEMON_OPTS || true
    echo "$NAME."
    ;;
  stop)
    echo -n "Stopping $DESC: "
    start-stop-daemon --stop --quiet --pidfile /var/run/$NAME.pid \
        --exec $DAEMON || true
    echo "$NAME."
    ;;
  restart|force-reload)
    echo -n "Restarting $DESC: "
    start-stop-daemon --stop --quiet --pidfile \
        /var/run/$NAME.pid --exec $DAEMON || true
    sleep 1
    start-stop-daemon --start --quiet --pidfile \
        /var/run/$NAME.pid --exec $DAEMON -- $DAEMON_OPTS || true
    echo "$NAME."
    ;;
  reload)
      echo -n "Reloading $DESC configuration: "
      start-stop-daemon --stop --signal HUP --quiet --pidfile /var/run/$NAME.pid
\
          --exec $DAEMON || true
      echo "$NAME."
      ;;
  status)
      status_of_proc -p /var/run/$NAME.pid "$DAEMON" nginx && exit 0 || exit $?
      ;;
  *)
    N=/etc/init.d/$NAME
    echo "Usage: $N {start|stop|restart|reload|force-reload|status}" >&2
    exit 1
    ;;
esac

exit 0
```

Make the init script executable and add nginx to the start-up applications.

```
sudo chmod +x /etc/init.d/nginx
sudo update-rc.d -f nginx defaults
```

Start nginx as a service

```
sudo service nginx start
```

## Create the deployment directory

Create the directory

```
sudo mkdir /var/www/
```

Give the staff group rights to the directory.

```
sudo chgrp -R staff /var/www
sudo chmod -R 755 /var/www
```

Make the [user] owner of the directory

```
sudo chown -R [user] /var/www
```

## Install ruby [RAILS]

Install rbenv globally

```
cd /usr/local
git clone git://github.com/sstephenson/rbenv.git rbenv
sudo chgrp -R staff rbenv
sudo chown -R [user] rbenv
sudo chmod -R 755 rbenv
```

Add rbenv to the bashrc script for the [user]

```
echo 'export RBENV_ROOT=/usr/local/rbenv' >> /home/[user]/.bashrc
echo 'export PATH="/usr/local/rbenv/bin:$PATH"' >> /home/[user]/.bashrc
echo 'eval "$(rbenv init -)"' >> /home/[user]/.bashrc
```

Install ruby-build

```
mkdir -p /usr/local/rbenv/plugins
cd /usr/local/rbenv/plugins
git clone git://github.com/sstephenson/ruby-build.git
```

Install ruby (this can take a while)

```
rbenv install 2.2.2; rbenv rehash
rbenv global 2.2.2
```

Install bundler

```
gem install bundler --no-rdoc --no-ri; rbenv rehash
```

## Install NodeJS [NodeApps/RAILS]

```
sudo add-apt-repository ppa:chris-lea/node.js
sudo apt-get update
sudo apt-get install nodejs
```

## Install ImageMagick[RAILS imageupload]

```
sudo apt-get install imagemagick --fix-missing
sudo ln -s /usr/bin/identify /usr/local/bin/identify
```

## Install Postgres [OPTIONAL]

This step may fail so try again if it does:

```
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib libpq-dev
```

Configuring local postgres access without providing password using trust method
Edit /etc/postgresql/[version]/main/pg_hba_conf.

```
local   all             postgres                                trust

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     trust
# IPv4 local connections:
host    all             all             127.0.0.1/32            trust
# IPv6 local connections:
host    all             all             ::1/128                 trust
```

Restart postgres server:

```
sudo pg_ctlcluster [9.3] main stop
sudo pg_ctlcluster [9.3] main start
sudo su - postgres
createuser [user] -s
```

Test if you can access the database as [user]:

```
psql -l // defaults to logged in ssh user
```

## Install MongoDB [OPTIONAL]

Add the key for the repository.

```
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
```

Create the /etc/apt/sources.list.d/mongodb.list file.

```
echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' |
sudo tee /etc/apt/sources.list.d/mongodb.list
```

Reload the repository.

```
sudo apt-get update
```

Install the package.

```
sudo apt-get install mongodb-10gen
```

## Installing redis [OPTIONAL]

```
cd ~
wget http://download.redis.io/redis-stable.tar.gz
tar xvzf redis-stable.tar.gz
cd redis-stable
make
sudo cp src/redis-server /usr/local/bin/
sudo cp src/redis-cli /usr/local/bin/
cd utils
sudo ./install_server.sh
cd ~
rm -rf redis-stable*
```

Test if redis is installed succesfully

```
redis-cli
```
