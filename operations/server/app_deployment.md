#Setting up an application for Capistrano deployment

## Setup SSH keys

Run the following command to check if the ssh agent has access to your key

```
ssh-add -L
```

If this command does not list your key, run the following command

```
ssh-add
```

Add an entry to ~/.ssh/config to allow agent forwarding for the host

```
Host example.com
  ForwardAgent yes
```

## Add required gems to your Gemfile

Add the following entries to your Gemfile:

```
gem 'dotenv-rails'

group :assets do
  gem 'turbo-sprockets-rails3'
end

group :development do
  gem 'capistrano-rbenv', '~> 2.0', require: false
  gem 'capistrano-bundler', '~> 1.1', require: false
  gem 'capistrano-rails',   '~> 1.1', require: false
  gem 'capistrano3-nginx_unicorn'
  gem 'capistrano-file-permissions'
end
```

Make sure you are not using the 12factor gem as it messes up the logging of
unicorn!

Run:

```
bundle install
```

## Add the Capfile

Add a file named 'Capfile' to the root of your project. Paste the following
contents in the file:

```
require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/rbenv'
require 'capistrano/bundler'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
require 'capistrano3/nginx_unicorn'
require 'capistrano/file-permissions'

Dir.glob('lib/capistrano/tasks/*.cap').each { |r| import r }
```

## Add the capistrano configuration

Add a file named 'deploy.rb' to the /config directory of your project. Paste the
following contents in the file (be sure to replace the application name, git
repo and deployment directory):

```
set :application, "[APP_NAME_EXAMPLE]"

set :log_level, :info

set :scm, :git
set :repo_url,  "git@github.com:[user]/example.git"
set :deploy_to, "/var/www/[APP_NAME_EXAMPLE]"
set :user, "[user]"
set :keep_releases, 5

set :ssh_options, {
  forward_agent: true,
  port: [PORT]
}

set :rbenv_type, :system
set :rbenv_ruby, '2.1.5'
set :rbenv_prefix, "RBENV_ROOT=#{fetch(:rbenv_path)}
RBENV_VERSION=#{fetch(:rbenv_ruby)} #{fetch(:rbenv_path)}/bin/rbenv exec"
set :rbenv_map_bins, %w{rake gem bundle ruby rails}
set :rbenv_roles, :all

SSHKit.config.command_map[:rake]  = "bundle exec rake"
SSHKit.config.command_map[:rails] = "bundle exec rails"

set :linked_files, %w{.env}
set :linked_dirs, %w{bin log tmp public/assets public/sites public/system}

set :file_permissions_roles, :all
set :file_permissions_paths, ["/usr/local/rbenv"]
set :file_permissions_users, ["[USER]"]
set :file_permissions_chmod_mode, "0770"

after "deploy:updated", "deploy:set_permissions:chmod"
```

Add a subdirectory named 'deploy' to the /config directory of your project.
Add a subdirectory named 'templates' to the newly created deploy directory.

Add a file named 'production.rb' to the /config/deploy directory of your
project. Paste the following contents in the file (be sure to replace the server
name and certificates):

```
set :stage, :production

server "example.com", user: "[USER]", roles: %w{web app db}
set :branch, "station"
set :nginx_server_name, "example.com"

set :nginx_use_spdy, false
set :nginx_ssl_certificate, "example.pem"
set :nginx_ssl_certificate_key, "example.key"

set :nginx_enable_pagespeed, true
set :nginx_pagespeed_enabled_filters, "lazyload_images"

set :unicorn_workers, 2
```

## Add the nginx_conf template

Create this file in `config/deploy/templates/nginx_conf.erb`

```
upstream unicorn_<%= fetch(:application) %> {
  server unix:/tmp/unicorn.<%= fetch(:application) %>.sock fail_timeout=0;
}

server {
  server_name www.<%= fetch(:nginx_server_name) %>;
  return 301 $scheme://<%= fetch(:nginx_server_name) %>$request_uri;
}
<% if fetch(:nginx_use_spdy) && fetch(:nginx_enable_pagespeed) %>
server {
  listen 80;
  server_name localhost;
  
  root <%= current_path %>/public;

  try_files $uri @unicorn_<%= fetch(:application) %>;

  location @unicorn_<%= fetch(:application) %> {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://unicorn_<%= fetch(:application) %>;
  }

  pagespeed off;
}
<% end %>
server {
<% if fetch(:nginx_use_spdy) %>
  listen 443 default ssl spdy;
  listen 80;

  ssl_certificate /etc/ssl/certs/<%= fetch(:nginx_ssl_certificate) %>;
  ssl_certificate_key /etc/ssl/private/<%= fetch(:nginx_ssl_certificate_key) %>;

  ssl_session_cache shared:SSL:10m;
  ssl_session_timeout 10m;
  ssl_prefer_server_ciphers on;
  ssl_ciphers ECDH+AESGCM:ECDH+AES256:ECDH+AES128:!ADH:!AECDH:!MD5;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_stapling on;
  ssl_stapling_verify on;

  resolver 8.8.8.8 8.8.4.4;
        
  add_header Strict-Transport-Security "max-age=31536000";
<% else %>
  listen 80 default;
<% end %>
  server_name <%= fetch(:nginx_server_name) %>;
  root <%= current_path %>/public;
  try_files $uri @unicorn_<%= fetch(:application) %>;

  location @unicorn_<%= fetch(:application) %> {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    <% if fetch(:nginx_use_spdy) %>proxy_set_header X-Forwarded-Proto https;<% end %>
    proxy_pass http://unicorn_<%= fetch(:application) %>;
    
    access_log <%= shared_path %>/log/nginx.access.log;
    error_log <%= shared_path %>/log/nginx.error.log;
  }
  <% if fetch(:nginx_enable_pagespeed) %>
  location ~ "\.pagespeed\.([a-z]\.)?[a-z]{2}\.[^.]{10}\.[^.]+" { add_header "" ""; }
  location ~ "^/ngx_pagespeed_static/" { }
  location ~ "^/ngx_pagespeed_beacon$" { }
  location /ngx_pagespeed_statistics { allow all; }
  location /ngx_pagespeed_global_statistics { allow all; }
  location /ngx_pagespeed_message { allow all; }
  location /pagespeed_console { allow all; }

  pagespeed EnableFilters <%= fetch(:nginx_pagespeed_enabled_filters)%>;
  <% if fetch(:nginx_use_spdy) %>pagespeed MapOriginDomain "http://localhost" "https://<%= fetch(:nginx_server_name)%>";<% end %>
  pagespeed Statistics on;
  pagespeed StatisticsLogging on;
  pagespeed LogDir /var/log/pagespeed;
  <% end %>
  client_max_body_size 4G;
  keepalive_timeout 10;

  error_page 500 502 504 /500.html;
  error_page 503 @503;

  location = /50x.html {
    root html;
  }

  location = /404.html {
    root html;
  }

  location @503 {
    error_page 405 = /system/maintenance.html;
    if (-f $document_root/system/maintenance.html) {
      rewrite ^(.*)$ /system/maintenance.html break;
    }
    rewrite ^(.*)$ /503.html break;
  }

  if ($request_method !~ ^(GET|HEAD|PUT|POST|DELETE|OPTIONS)$ ){
    return 405;
  }

  if (-f $document_root/system/maintenance.html) {
    return 503;
  }

  location ~ \.(php|html)$ {
    return 405;
  }
}
```

## Add the unicorn configuration template

Create this file as `config/deploy/templates/unicorn.rb.erb`

```
user "[user]"
working_directory "<%= current_path %>"
pid "<%= fetch(:unicorn_pid) %>"
stderr_path "<%= fetch(:unicorn_log) %>"
stdout_path "<%= fetch(:unicorn_log) %>"

listen "/tmp/unicorn.<%= fetch(:application) %>.sock"
worker_processes <%= fetch(:unicorn_workers) %>
timeout 30

preload_app true

before_exec do |server|
  ENV["BUNDLE_GEMFILE"] = "<%= current_path %>/Gemfile"
end

before_fork do |server, worker|

  # Quit the old unicorn process
  old_pid = "#{server.config[:pid]}.oldbin"
  if File.exists?(old_pid) && server.pid != old_pid
    begin
      Process.kill("QUIT", File.read(old_pid).to_i)
    rescue Errno::ENOENT, Errno::ESRCH
      # someone else did our job for us
    end
  end

  sleep 1
end
```

## Add the unicorn initialisation template

Create this file as `config/deploy/templates/unicorn_init.erb`

```
#!/bin/bash
### BEGIN INIT INFO
# Provides: unicorn
# Required-Start: $remote_fs $syslog
# Required-Stop: $remote_fs $syslog
# Default-Start: 2 3 4 5
# Default-Stop: 0 1 6
# Short-Description: Manage unicorn server
# Description: Start, stop, restart unicorn server for a specific application.
### END INIT INFO
set -e

# Feel free to change any of the following variables for your app:
TIMEOUT=${TIMEOUT-60}
APP_ROOT=<%= current_path %>
PID=<%= fetch(:unicorn_pid) %>

CMD="cd <%= current_path %>; /usr/local/rbenv/shims/bundle exec unicorn -D -c <%= fetch(:unicorn_config) %> -E <%= fetch(:stage) %>"

set -u

OLD_PIN="$PID.oldbin"

sig () {
  test -s "$PID" && kill -$1 `cat $PID`
}

oldsig () {
  test -s $OLD_PIN && kill -$1 `cat $OLD_PIN`
}

run () {
  eval $1
}

case "$1" in
start)
  sig 0 && echo >&2 "Already running" && exit 0
  run "$CMD"
  ;;
stop)
  sig QUIT && exit 0
  echo >&2 "Not running"
  ;;
force-stop)
  sig TERM && exit 0
  echo >&2 "Not running"
  ;;
restart|reload)
  sig USR2 && echo reloaded OK && exit 0
  echo >&2 "Couldn't reload, starting '$CMD' instead"
  run "$CMD"
  ;;
upgrade)
  if sig USR2 && sleep 2 && sig 0 && oldsig QUIT
  then
    n=$TIMEOUT
    while test -s $OLD_PIN && test $n -ge 0
    do
      printf '.' && sleep 1 && n=$(( $n - 1 ))
    done
    echo

    if test $n -lt 0 && test -s $OLD_PIN
    then
      echo >&2 "$OLD_PIN still exists after $TIMEOUT seconds"
      exit 1
    fi
    exit 0
  fi
  echo >&2 "Couldn't upgrade, starting '$CMD' instead"
  run "$CMD"
  ;;
reopen-logs)
  sig USR1
  ;;
*)
  echo >&2 "Usage: $0 <start|stop|restart|upgrade|force-stop|reopen-logs>"
  exit 1
  ;;
esac
```

## Commit your work

Commit your work.

```
git add --all
git commit -m 'Add capistrano configuration'
git push origin master
```

## Deploy

Deploy the application to the server.

```
cap production deploy
```

The deployment will fail because your .env file is missing.

## Add .env file

Add a file and set the required entries (depends on specific project)

```
nano /var/www/[APP_NAME]/shared/.env
```

## Redeploy

Redeploy the application to the server.

```
cap production deploy
```
