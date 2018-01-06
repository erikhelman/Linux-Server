# Udacity Linux Server Deployment Project

The purpose of this project is to setup, update and configure a web server and database on a new Linux server and then successfully deploy a web application to that server.

The web application uses Python, Flask, mod_wsgi, Apache and Postgres. A detailed summary of the setup is below.

Server IP - 52.15.114.221

Website URL - http://52.15.114.221/

## Create the Server

This project is using Amazon Lightsail.
1. First, log in to [Amazon Web Services](https://aws.amazon.com/)
1. Select Lightsail from the services list
1. Select Create instance
1. For platform, select Linux/Unix
1. For blueprint, select "OS Only" and Ubuntu
1. Give the instance a name and click Create
1. Click on the server and click Connect Using SSH to access the command line

## Install and update all required software

First, set the timezone to UTC

    sudo dpkg-reconfigure tzdata

From the menu, select "None of the Above", then "UTC".
Confirmation should be displayed on the screen

Current default time zone: 'Etc/UTC' <br>
Local time is now:      Thu Jan  4 15:08:11 UTC 2018. <br>
Universal Time is now:  Thu Jan  4 15:08:11 UTC 2018. <br>

Update all installed packages and install required software

    sudo apt-get update
    sudo apt-get install apache2 python3-pip git libapache2-mod-wsgi-py3
    sudo apt-get upgrade
    sudo apt-get dist-upgrade

A message will appear while updating that indicates a new version of /boot/grub/menu.lst is available, select "Keep the local version currently installed".

Then restart the server

    sudo reboot


## Create the grader user and their key

Create the user

    sudo adduser grader

Add the user to the sudoers list

    sudo touch /etc/sudoers.d/grader
    sudo nano /etc/sudoers.d/grader

And add the following line

    grader ALL=(ALL) NOPASSWD:ALL

Create a new key pair on your local machine

    ssh-keygen

Switch to the grader user on the remote server

    su - grader

Create a .ssh folder in the home folder with a file called authorized_keys

    touch .ssh/authorized_keys

Edit the authorized keys file. Copy the contents from the local machine .pub file to the file on the server. Then change the permissions of the file and folder on the server.

    chmod 700 .ssh
    chmod 644 .ssh/authorized_keys

Connect as the grader user

    ssh -i [keyfile] grader@[ip]

## Update SSH configuration

Update the ssh config file

    sudo nano /etc/ssh/sshd_config

Change PermitRootLogin to no, update the port from 22 to 2200 and confirm PasswordAuthentication is set to no

Restart the ssh service

    sudo service ssh restart

Port and the next section's firewall changes must also be made in the Lightsail UI. In the Lightsail console, select Networking. Add a Custom rule for TCP on port 2200 and a Custom rule for UDP on port 123. The HTTP rule on port 80 should already exist.

Now the grader account can be connected by specifying the key file and port

    ssh -i [keyfile] grader@[ip] -p 2200

## Update firewall settings

Run the following commands to allow all outgoing traffic but only incoming traffic for NTP, HTTP and SSH (now on port 2200)

    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow 2200/tcp
    sudo ufw allow 123/udp
    sudo ufw allow 80/tcp
    sudo ufw enable

## Clone the Catalog app repository and configure Apache

Change to the /var/www directory <br>
Create a new application directory and change the owner to the grader account

    sudo mkdir catalog
    sudo chown -R grader:grader catalog

Clone the catalog project from github

    git clone git@github.com:erikhelman/catalog.git catalog

Create a catalog.wsgi file in the catalog folder and enter the following

```
#!/usr/bin/python
import sys
import logging

sys.path.insert(0,"/var/www/catalog/")

from catalog import app as application
```

Install virtualenv

    cd catalog
    pip3 install virtualenv

Create a virtualenv

    virtualenv venv

Activate the virutalenv and install the project requirements

    . venv/bin/activate
    pip3 install -r requirements.txt

Create a new config file

    sudo nano /etc/apache2/sites-available/catalog.conf

And add the following

```
<VirtualHost *:80>
  ServerName 52.15.114.221  
  ServerAdmin webmaster@localhost

  WSGIDaemonProcess catalog home=/var/www/catalog python-home=/var/www/catalog/venv
  WSGIProcessGroup catalog
  WSGIApplicationGroup %{GLOBAL}
  WSGIScriptAlias / /var/www/catalog/catalog.wsgi

  <Directory /var/www/catalog/catalog/>
    Order allow,deny
    Allow from all
  </Directory>

  Alias /static /var/www/catalog/static
  <Directory /var/www/catalog/static/>
    Order allow,deny
    Allow from all
  </Directory>

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined

</VirtualHost>
````

Enable the virtual host

    sudo a2ensite catalog

Restart apache

    sudo service apache2 restart


## Configure the database

Install postgres

    sudo apt-get install postgresql postgresql-contrib

Switch to the postgres user and create the catalog user and database

    sudo su - postgres
    psql
    create database catalog;
    create user catalog with password 'catalog';
    grant all privileges on database catalog to catalog;

Modify catalog.py in /var/www/catalog

    postgresql://catalog:catalog@localhost/catalog

Run the setup for the catalog application (detailed in the catalog README) to create the tables and initial data

## Reference documentation

Udacity course materials <br>
[Postgres](https://help.ubuntu.com/community/PostgreSQL) <br>
[Mod_wsgi](http://modwsgi.readthedocs.io/en/develop/)
