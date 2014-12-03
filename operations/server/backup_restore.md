estoring backups

Visit:
[https://console.aws.amazon.com/s3/home?region=eu-west-1](https://console.aws.amazon.com/s3/home?region=eu-west-1)

* Go to your backups bucket.
* Click the project you want a backup from.
* Make sure you take the latest backups for restoring data.
* Download de tar package.

We need to move the backup to the remote server using scp:

```
cd ~/Downloads
scp -P 12013 [project].tar [user]@[server]:
```

The tar file should be on the remote server.

Login to remote server using ssh:

```
ssh [user]@[ip-address] -p [port]
```

Issue following commands to check if the file got copied.

```
cd ~
ls
```

If the file is present we should untar it with the following command:

```
tar -xvf [project].tar
```

You should now see:

```
[project]
|- archives
|- databases
```

Inside archives is a www.tar.gz file we need extracted and navigated to:

```
tar -xvzf www_archive.tar.gz
cd var/www/[project]/shared/public
```

Inside the public folder should be the public assets served from the backup.

Copy the necessary public files to the public files on the server using:

```
mv
#or
cp
```

Now that the public assets are restored we still need to handle the database.

```
cd ~/var/databases
```

Postgres:

```
psql --dbname=[project]_production -f PostgreSQL.sql
```

MongoDB:

```
cd ~/[project]/databases/MongoDB/[project]_production
mongorestore .
```


Don't forget to cleanup afterwards:

```
cd ~
rm -rf [project backup folder]
```
