# Setting up a virtual server from scratch

##Prerequisites

Virtual private server

* [Digital Ocean](https://www.digitalocean.com/)

Connect to the server using `ssh` by using the password sent in mail:

```
ssh root@[ip-address]
```

## Set the locale settings

Add the following line to to `/etc/environment` and reboot:

```
LC_ALL="en_US.utf8"
```

## Update the server

```
sudo apt-get update
sudo apt-get upgrade
```

## Configure NTP

Configure the correct time zone (Europe/Brussels).

```
sudo dpkg-reconfigure tzdata
```

Install the ntp service.

```
sudo apt-get install ntp
```

Configure the servers.

```
sudo nano /etc/ntp.conf  
```

Replace the existing servers with the following.

```
server 0.be.pool.ntp.org  
server 1.be.pool.ntp.org  
server 2.be.pool.ntp.org  
server 3.be.pool.ntp.org
```

Restart the ntp service.

```
sudo service ntp restart
```

Check if everything is set up correctly.

```
ntpq -p
```

## Enable automatic updates

Enable automatic updates by configuring the unattended upgrades package.

```
sudo apt-get install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

Select 'Yes'.


## Secure the server

Create a new user that will be used to access/deploy software to the machine.

```
sudo useradd -g staff -s /bin/bash -d /home/[user] -m [user]
```

Change the password.

```
passwd [user]
```

Give the created user super user priviliges and disable asking for a password
for sudo actions

```
sudo nano /etc/sudoers
```

Add this line to the end of the file

```
[user] ALL=(ALL) NOPASSWD:ALL
```

Add the SSH authorized keys to the list of authorized keys on the server.

```
mkdir /home/[user]/.ssh
chown -R [user] /home/[user]/.ssh
exit
cat ~/.ssh/id_rsa.pub | ssh [user]@[ip-address] "cat >>
~/.ssh/authorized_keys"
```

Make sure to add the ssh keys of your team members with the following command:

```
echo "[Public RSA key team member]" | ssh [user]@[ip-address] "cat >>
~/.ssh/authorized_keys"
```



Copy the contents of the `autorized_keys` to `~/.ssh/authorized_keys` on the
server.

```
ssh [user]@[ip-address]
nano ~/.ssh/authorized_keys
```

Change the following lines in `/etc/ssh/sshd_config`:

(This changes the default SSH port to `[port]`, disables password over SSH and
disables root login over SSH)

```
Port [port]
PermitRootLogin no
PasswordAuthentication no
```

Restart the SSH server.

```
sudo service ssh restart
```

Configure the firewall. (Be careful, if you make a mistake in the next steps,
you won't be able to reconnect to the server (!!!))

```
sudo apt-get install ufw  
sudo ufw default deny incoming  
sudo ufw default allow outgoing  
sudo ufw allow 12013/tcp  
sudo ufw allow 80/tcp  
sudo ufw allow 443/tcp
sudo ufw enable  
```

From your own machine now connect using the following command.

```
ssh [user]@[ip-address] -p [port]
```

If you did everything in the guide, you should be able to connect without
providing a password.


## [OPTIONAL] Configure swap

Create a new swap file

```
sudo dd if=/dev/zero of=/swapfile bs=1024 count=1024k
```

Activate the swap file

```
sudo mkswap /swapfile
```

Add the following line to `/etc/fstab` to make sure it survives a reboot

```
/swapfile none swap sw 0 0
```

Configure a low swappiness setting:

```
echo 10 | sudo tee /proc/sys/vm/swappiness
echo vm.swappiness = 10 | sudo tee -a /etc/sysctl.conf
```

Set proper permissions on the swap file:

```
sudo chown root:root /swapfile  
sudo chmod 0600 /swapfile
```
