#Creating and configuring ssl.

##Creating a `.pem`-file:

Paste the entire body of each certificate into one text file in the following
order:

- The Primary Certificate
- All Intermediate Certificates
- The Root Certificate

Make sure to include the beginning and end tags on each certificate. The result
should look like this:

```
-----BEGIN CERTIFICATE-----
(Your Primary SSL certificate: your_domain_name.crt)
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
(Your Intermediate certificate: intermediate.crt)
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
(Your Root certificate: root.crt)
-----END CERTIFICATE-----
```

Save the combined file as `your_domain_name.pem`.

## Add ssl-certificate and private key to server

Use the command below to copy files to the server:

```
scp -P 12013 path/to/file [user]k@[ip_address]:path/to/file
```

Copy both certificate and key to `~/filename`

Move the `.pem`-file into `/etc/ssl/certs`

```
sudo mv your_domain_name.pem /etc/ssl/certs
```

Move the private key into `/etc/ssl/private`

```
sudo mv your_domain_name.key /etc/ssl/private
```

## Make sure only the root user has access to the private key

Set user permissions for the certificate and key with following commands:

certificate:
```
sudo chmod 755 /etc/ssl/certs/certificate_name.pem
```

key:
```
sudo chmod 700 /etc/ssl/private/key_name.key
```

## Edit the nginx configuration:

Add the ssl-certificate and ssl-certificate-key to `deploy/production.rb`

```
set :nginx_ssl_certificate, "your_domain_name.pem"
set :nginx_ssl_certificate_key, "your_domain_name.key"
```
## Redeploy

Redeploy the app with spdy enabled:

```
cap production deploy
```
