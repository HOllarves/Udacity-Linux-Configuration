# Udacity Linux Server Configration

## Prerequisites

For this submission I will be providing a private key to log in to my amazon lightsail server. This key will be sent in the reviewer's notes.

## Goals

### #1 - Virtual Machine

Using Amazon Lightsail I started my own ubuntu server.

Public IP: 35.176.192.35

### #2 - Loggin in with SSH

Amazon Lightsail allow its users to use an interactive terminal on their site to directly communicate with their servers using ssh. However, I always prefer to use my own ssh client runnning on my personal ubuntu machine.

Initially, to enter my server using ssh I created a .pem key file using amazon lightsail client. I saved this .pem file to my .ssh folder, and then simply logged in using `ssh ubuntu@35.176.192.35 -i udacityServerKey.pem`

### #3 - Set up

First, I updated all my packages.

`sudo apt-get update`

`sudo apt-get upgrade -y`

Then, as a personal preference, I installed the automatic security upgrades for Ubuntu. It not only makes my server more safe, it also brings useful tools.

`sudo apt-get install unattended-upgrades`

`sudo dpkg-reconfigure -plow unattended-upgrades`

### #4 - Add user

To add the grader user I simply:

`adduser grader`

Then I gave sudo permission to it:

`echo "grader ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/grader`

### #5 - Set up SSH for grader

I generated a SSH key-pair for grader:

```
#On my local machine

ssh-keygen -t rsa
```

I then configured this ssh key-pair in my Lightsail server

`su grader`

`mkdir ~/.ssh`

`vim ~/.ssh/authorized_keys` -> Copied the authorized key

Later, I configured permissions for the files I just created:

`chmod 700 ~/.ssh`

`chmod 644 ~/.ssh/authorized_keys`

From now on, the grader can log in using:

`ssh grader@35.176.192.35 -i id_rsa_udacity`

### #6 - Change SSH port

I opened the ssh config file

`vim /etc/ssh/sshd_config` (make sure not to use ssh_config)

Changed port declaration from 22 to 2200

Also made sure to set:

`PermitRootLogin` to `no`,

`PasswordAuthentication` to `no` 

To avoid password authentication when login in, we want to use ssh key-pairs for that.

Then I simply restarted the ssh service `sudo service ssh restart`

NOTE: As I was using Amazon's Lightsail an extra step was needed. It is required to add the custom SSH port to the allowed ports table in the Amazon Lightsail client.

### #7 - Configuring UFW

To begin, I initialized some default rules

`sudo ufw default deny incoming` -> Deny all incoming requests

`sudo ufw default allow outgoin` -> Allow all outgoing requests

Then I allowed incoming connection from the selected SSH port:

`sudo ufw allow 2200/tcp`

HTTP requests:

`sudo ufw allow www`

And NTP requests:

`sudo ufw allow ntp`

Finally, I enabled ufw:

`sudo ufw enable`

From now on, the grader can log in using:

`ssh grader@35.176.192.35 -p 2200 -i id_rsa_udacity`

### #8 - Timezone configuration

Simply:

`sudo dpkg-reconfigure tzdata` -> Select none of the above and then UTC

### #9 - Configuring Apache to serve WSGI python app

First I installed the required packages:

`sudo apt-get install apache2`

`sudo apt-get install libapache2-mod-wsgi`

`sudo apt-get install postgresql`

I created a `app.wsgi` file on my project's root directory to point to my wsgi application:

```
import sys
import os
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/udacity-item-catalog/catalog")

from app import app as application
```

I then configured apache to serve it. In `etc/apache2/sites-enabled/000-default.conf`:

```
<VirtualHost *:80>

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined

	ServerName localhost
	ServerAlias ec2-35-176-192-35.eu-west-2.compute.amazonaws.com
	WSGIDaemonProcess catalog user=grader group=grader threads=5
	WSGIScriptAlias / /var/www/udacity-item-catalog/catalog/app.wsgi

	<Directory /var/www/udacity-item-catalog/catalog>
		WSGIProcessGroup catalog
		WSGIApplicationGroup %{GLOBAL}
		Order deny,allow
		Allow from all
	</Directory>

</VirtualHost>

```

`WSGIScriptAlias / /var/www/udacity-item-catalog/catalog/app.wsgi` points to my app.wsgi file. Allowing apache to serve it

`WSGIDaemonProcess catalog user=grader group=grader threads=5` starts a process to serve my application

`ServerAlias ec2-35-176-192-35.eu-west-2.compute.amazonaws.com` allows my server to use it's own domain name (Required for Google's OAuth configuration)

### #10 - Database set up

First, I logged in as the postgres user:

`sudo -i -u postgres`

Then I created a database user and a database:

`createuser --interactive -P` -> This will start an interactive console program to create a new postgres user

Finally I created my catalog database:

- To run postgres console: `psql`

- To create database: `CREATE DATABASE catalog;`

### #11 - Set up of catalog app

First I installed git:

`sudo apt-get install git`

Then I simply cloned my repo to the `/var/www/` directory

It is also important to protect my .git folder so I used:

`sudo chmod 700 /var/www/udacity-item-catalog/catalog/.git` -> this to avoid unauthorized modifications to my git configuration files

Later, I installed my application dependencies:

```
sudo apt-get install python-flask
sudo apt-get install python-sqlalchemy
sudo apt-get install python-pip
sudo pip install bleach
sudo pip install flask-seasurf
sudo pip install github-flask
sudo pip install httplib2
sudo pip install oauth2client
sudo pip install requests
sudo pip install faker #only for seeders
```

Finally I restarted the apache service:

`sudo service apache2 restart`

### #12 - OAuth

After adding my server alias to my `000-default.conf` it is important to enable the virtual host using `sudo a2ensite`

Then simply I went to Google Cloud Console -> Udacity Item Catalog -> APIs -> Credentials -> Web OAuth Credentials  and added `ec2-35-176-192-35.eu-west-2.compute.amazonaws.com` to my Javascript Origins and `ec2-35-176-192-35.eu-west-2.compute.amazonaws.com/gconnect` to my authorized callbacks






