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

## Add the unicorn configuration templates

To be added

## Add the nginx configuration templates

To be added

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
