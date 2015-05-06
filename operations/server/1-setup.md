# Setting up a virtual server from scratch

##Prerequisites

Providers:

* [Digital Ocean](https://www.digitalocean.com/)

Connect to the server using `ssh`:

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

Configure the correct time zone

```
sudo dpkg-reconfigure tzdata
```

Install the ntp service.

```
sudo apt-get install ntp
```

Configure the network time protocol for your zone.
Find a ntp server for your region on: [http://www.pool.ntp.org/en/](http://www.pool.ntp.org/en/)

```
sudo nano /etc/ntp.conf
```

Replace the existing servers with the following.

```
# Replace with your zone NTP servers
# Zone: Belgium
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

Give the user, super user priviliges and disable asking for a password on  sudo actions

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
```

Secure the server by ssh'ing over your public ssh key.
If you haven't created ssh keys yet take a look at: [Generate ssh keys](https://help.github.com/articles/generating-ssh-keys/)

```
cat ~/.ssh/id_rsa.pub | ssh [user]@[ip-address] "cat >> ~/.ssh/authorized_keys"
```

Make sure to add the ssh keys of your team members with the following command:

```
echo "[Public RSA key team member]" | ssh [user]@[ip-address] "cat >> ~/.ssh/authorized_keys"
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
sudo ufw allow [port]/tcp  
# Skip this if you aren't going to host any websites or apps
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

## [OPTIONAL] Configure swap space

### What is swap space?

Swap space in Linux is used when the amount of physical memory (RAM) is full. If
the system needs more memory resources and the RAM is full, inactive pages in
memory are moved to the swap space. While swap space can help machines with a
small amount of RAM, it should not be considered a replacement for more RAM.
Swap space is located on hard drives, which have a slower access time than
physical memory.

Swap should equal 2x physical RAM for up to 2 GB of physical RAM, and then an
additional 1x physical RAM for any amount above 2 GB, but never less than 32 MB.

Using this formula, a system with 2 GB of physical RAM would have 4 GB of swap,
while one with 3 GB of physical RAM would have 5 GB of swap. Creating a large
swap space partition can be especially helpful if you plan to upgrade your RAM
at a later time.

### What should we use it for?

Mainly to run RAM intensive applications on low memory. For example a rails
application will approx. take about 700-800 MB of RAM to run properly. Hosting a
rails application typically involves unicorn webserver that allows zero down
time of an app by spawning unicorn workers.

The problem is unicorn won't be able to spawn another worker because there is
insufficient RAM available. This is where SWAP space comes in as temporary
solution.

A new worker can be created and it takes the place of the original unicorn process
allowing for zero down time.

## Configuring swapspace

Note: 

This formula is calculated on a 1GB RAM VPS:

```
1GB RAM = 1GB SWAP = 1024k
```

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
