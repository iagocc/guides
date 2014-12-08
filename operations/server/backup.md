#Configuring backups

## Installation

Log in to the remote server:

```
ssh [user]@[ip-address] -p [port]
```

Backup v4.x is distributed using RubyGems.
[http://meskyanichi.github.io/backup/v4/](http://meskyanichi.github.io/backup/v4/)

To install the latest version, run:

```
gem install backup whenever --no-ri --no-rdoc
```

Test and see if the command backup and whenever can be run.
If not, issue the following command:

```
rbenv rehash // Recreates shims
```

!!! Do not add gem backup to another application's Gemfile. !!!

This will install Backup, along with all of it's required dependencies.

## Updating

To update Backup to the latest version, run:

```
gem install backup
```

If you wish to install a specific version of Backup, you can specify the version
as follows:

```
gem install backup -v '4.1.0'
```

When you update Backup, the new version of the Backup gem will be installed, but
older versions are not removed.
To cleanup, run:

```
gem cleanup backup
```

## Getting started

Setting up global backup config. This file will be loaded first when accessing
the backup gem. Run:

```
backup generate:config
```

Let's generate a global Backup model file:

```
backup generate:model --trigger global_backup \
  --archives --databases="mongodb, postgresql, redis" --storages="s3" \
  --compressor="gzip" --notifiers="slack"
```

Some cloud storage solutions have a limit on the upload size. If this is the
case we could use the following option: 

```
[--splitter]                 # Add Splitter to the model
```

If your final backup package is global_backup.tar.gz and is 1 GB in size, it
would split this file into:

```
global_backup.tar.gz.aaa
global_backup.tar.gz.aab
global_backup.tar.gz.aac
global_backup.tar.gz.aad
```


For critical and sensitive projects you may want to add the encryption options
like so:

```
[--encryptor=ENCRYPTOR]      # (gpg, openssl)
```

This will create a new file: ~/Backup/models/global_backup.rb
Open the file and just omit what you don't need, and change what you do need and
you're done.


```
# encoding: utf-8

##
# Backup Generated: global_backup
# Once configured, you can run the backup with the following command:
#
# $ backup perform -t global_backup [-c <path_to_configuration_file>]
#
# For more information about Backup's components, see the documentation at:
# http://meskyanichi.github.io/backup
#
# DECLARE THE PROJECT_MODEL NAME HERE
Model.new(:[server-name], 'Create a backup of all apps and databases on the server')
do

  archive :www_archive do |archive|
    # Run the `tar` command using `sudo`
    #archive.use_sudo
    archive.add "/var/www"
    #archive.exclude "/path/to/a/excluded_folder"
  end

  ##
  # MongoDB [Database]
  #
  #database MongoDB do |db|
  #end

  ##
  # PostgreSQL [Database]
  #
  #database PostgreSQL do |db|
  #  To dump all databases, set `db.name = :all` (or leave blank)
  #  db.name               = :all
  #  db.username           = "[user]"
  #  db.password           = ""
  #  db.host               = "localhost"
  #  When dumping all databases, `skip_tables` and `only_tables` are ignored.
  #  db.skip_tables        = ["skip", "these", "tables"]
  #  db.only_tables        = ["only", "these", "tables"]
  #end

  ##
  # Redis [Database]
  #
  #database Redis do |db|
  # db.mode               = :copy # or :sync
  # Full path to redis dump file for :copy mode.
  # db.rdb_path           = '/var/lib/redis/6379/dump.rdb' # be sure to check
this PATH!!!
  # When :copy mode is used, perform a SAVE before
  # copying the dump file specified by `rdb_path`.
  # db.invoke_save        = false
  # db.host               = 'localhost'
  #  db.port               = 6379
  #  db.password           = ''
  #  db.additional_options = []
  #end

  ##
  # Amazon Simple Storage Service [Storage]
  #
  store_with S3 do |s3|
    s3.access_key_id     = "[S3_ACCESS_KEY_ID]"
    s3.secret_access_key = "[S3_SECRET_ACCESS_KEY]"
    s3.region            = "[S3_BUCKET_REGION]"
    s3.bucket            = "[BUCKET_NAME]"
    # Or, to use a IAM Profile:
    s3.use_iam_profile = false
    #s3.path               = ''
    s3.keep               = 5
  end

  compress_with Gzip

  ##
  # Slack [Notifier]
  #
  notify_by Slack do |slack|
    slack.on_success = true
    slack.on_failure = true

    slack.team = '[TEAM_NAME]'
    slack.token = '[API_TOKEN]'

    slack.channel = '[#CHANNEL]'
    slack.username = '[BACKUP USERNAME]'
    slack.icon_emoji = ':floppy_disk:'
  end

end
```

## Performing backups

Before performing backups make sure the config files is configured correctly and
the following command succeeds:

```
backup check
```

The most basic command for performing a backup is:

```
backup perform --trigger [MODEL_NAME in global_backup.rb]
```

Advanced options:
[http://meskyanichi.github.io/backup/v4/performing-backups/](http://meskyanichi.github.io/backup/v4/performing-backups/)

## Utilities

If for whatever reason a utility used by backup can't be found through the which
command or a different version is used then needed. You can configure the paths
to these utilities through the utility model.

[http://meskyanichi.github.io/backup/v4/utilities/](http://meskyanichi.github.io/backup/v4/utilities/)

## Scheduling backups

Simple. Just use a cron task to invoke the Backup CLI.

An easy way to do this is using the whenever gem.

* Generate a schedule.rb file with the Whenever gem

```
cd ~/Backup
mkdir config # Whenever assumes a `config` directory exists
wheneverize
```

* Open the config/schedule.rb file and add the following:


```
every 1.day, :at => '2:00 am' do
  #Be sure to add the full shim path to backup command else it will not run.
  command "/usr/local/rbenv/shims/backup perform -t [servername from model] " 
end
```

* Run whenever with no arguments see the crontab entry this will create

```
whenever # This will NOT CREATE the crontab entry.
```

* To write (or update) this job in your crontab, use:

```
whenever --update-crontab
```

* To remove the crontab entry which you just added and replace it with one
  specifying the identifier

```
whenever --clear-crontab
```

* Remove crontab completely

```
crontab -e  # select editor to edit in
```
