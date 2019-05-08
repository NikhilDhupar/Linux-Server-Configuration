# Linux-Server-Configuration
---

#### Project Description
A baseline installation of a Linux server and prepare it to host web applications. Learning how to secure your server from a number of attack vectors, install and configure a database server, and deploy one of your existing web applications onto it.

## Server Hosting
To complete this project we need to host a linux server instance. This website is hosted using amazon EC2 services.
- Public IP Address - 52.90.89.231
- SSH port - 2200
- Website address to view live application - 52.90.89.231.xip.io

I have removed the instance so this ip is no longer working
## Resources Used on server to build website
  - Apache2 server
  - WSGI
  - Flask Framework

### Steps to host on AWS
- Sign in on aws and choose EC2 services
- Click on launch instance button and follow the steps to choose your desired server. I have used ubuntu 18.04. You might see the amazon documentation if needed.
- Download the ssh key provided in the process and save it to a safe location.
- Now that your instance is running its time to login to the server.
- Open terminal and use this command to login ` $ ssh  <Username>@<Public IP address> -i <privateKeyOfInstance> `
### Configuring aws console

On your aws console on left side click on security groups and create a security group with these 2 rules
 
- Custom TCP rule on port 2200
- HTTP rule on port 80

These 2 rules will help you access your server globally.

### Configuring Server
---
##### Update all currently installed packages.
```
$ sudo apt-get update
$ sudo apt-get upgrade
```
##### Change port from 22 to 2200
We configured aws server firewall before so if not done do that first else you will loose your machine
* Locate the line port 22 in the file /etc/ssh/sshd_config and edit it to port 2200.
* Use this command to view config file ` $ sudo vi /etc/ssh/sshd_config `
* Restart ssh services ` $ sudo service ssh restart `

##### Creating firewall
Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)

```
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw allow 2200/tcp
$ sudo ufw deny 22/tcp
$ sudo ufw enable
```
##### Creating a new user called grader, and generating a SSH key pair for grader
- Add new user - grader 
    `$ sudo adduser grader`
- Give grader sudo access 
    `$ sudo vi /etc/sudoers.d/grader`
- Add following line to this file
    `grader ALL=(ALL) NOPASSWD:ALL`
- Generate a keypair and push it to server.
    Always Use your local machine to generate a key pair to keep your key private
    `$ ssh-keygen -t rsa`
- Push it to server:
    Firstly we need to switch to newly created grader user and go to home directory. 
    ``` 
    $ su grader
    $ cd ~
    ```
    Create `.ssh` directory in home of server machine. Add a file named authorized_keys to hold your ssh key
    ```
    $ sudo mkdir .ssh
    $ sudo touch .ssh/authorized_keys
    ```
    Copy and paste the key from your local machine usign vi editor:
    ` $ sudo vi .ssh/authorized_keys `
    
    Changing permission of `.ssh` and `.ssh/authorized_keys`
    ```
    $ chmod 700 .ssh
    $ chmod 644 .ssh/authorized_keys
    ```
##### Now you are able to log into server using the grader account and port 2200.
`ssh grader@52.90.89.231 -p 2200 -i ~/.ssh/udacitynanosshkey`
##### Configure Timezone to UTC
Use this command to change timezone - ` $ sudo timedatectl set-timezone UTC `

### Deploy Flask project on server
---
##### 1. Install Apache
` sudo apt-get install apache2 `
##### 2. Install mod_wsgi
- Run `$ sudo apt-get install libapache2-mod-wsgi python-dev`
- Enable mod_wsgi with `$ sudo a2enmod wsgi`
- Start the web server with `$ sudo service apache2 start`
##### 3. Clone the Catalog app from Github
```
$ cd /var/www
$ sudo mkdir FlaskApp
$ cd FlaskApp/
$ sudo git clone https://github.com/NikhilDhupar/Item_Catelog.git
```
- Rename `catalog.py` file in Item_Catelog directory to `__init__.py`
    `$ sudo mv catalog.py __init__.py`
##### 4. Create FlaskApp.wsgi
- `$ sudo vi FlaskApp.wsgi`
- Insert the following content in this file
    ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/FlaskApp/")
    from Item_Catelog import app as application
    application.secret_key = 'supersecretkey'
    ```
##### 5. Create virtual environment
cd to Item_Catelog directory and follow these commands
- Install pip `$ sudo apt-get install python-pip`
- Install the virtual environment `$ sudo pip install virtualenv`
- Create a new virtual environment with `$ sudo virtualenv venv`
- Activate the virutal environment source `$ sudo venv/bin/activate`
- Change permissions `$ sudo chown -R grader:grader venv`
##### 6. Installing dependencies
Follow these commands to install the required dependencies
```
$ sudo apt-get install python-pip
$ sudo pip install Flask
$ sudo pip install sqlalchemy psycopg2 sqlalchemy_utils
$ sudo pip install httplib2 oauth2client requests
```
##### 7. Configure and enable a new virtual host
Run this command - `sudo vi /etc/apache2/sites-available/FlaskApp.conf`
Paste the following code into this file
```
<VirtualHost *:80>
    ServerName 52.90.89.231
    ServerAlias ec2-52-90-89-231.compute-1.amazonaws.com
    ServerAdmin nikhil171521.cse@chitkara.edu.in
    WSGIScriptAlias / /var/www/FlaskApp/FlaskApp.wsgi
    <Directory /var/www/FlaskApp/Item_Catelog/>
            Order allow,deny
            Allow from all
    </Directory>
    Alias /static /var/www/FlaskApp/Item_Catelog/static
    <Directory /var/www/FlaskApp/Item_Catelog/static/>
            Order allow,deny
            Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
Enable the virtual host - `$ sudo a2ensite FlaskApp.conf`

##### 8. Install and configure PostgreSQL
Follow these commands to install PostgreSQL
```
$ sudo apt-get install libpq-dev python-dev
$ sudo apt-get install postgresql postgresql-contrib
```
Creating a new database for Flask application named nikhil
```
$ sudo -i -u postgres
$ psql
# CREATE USER nikhil WITH PASSWORD 'admin123';
# CREATE DATABASE nikhil;
# GRANT ALL PRIVILEGES ON DATABASE nikhil TO nikhil;
# \q;
$ exit
```
Change database connection in `__init__.py` to `engine = create_engine('postgresql://nikhil:<password>@localhost/nikhil') `
##### 9. Restart apache server
`$ sudo service apache2 restart`

---
## Refrences
- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps#step-two-%E2%80%93-creating-a-flask-app
- https://www.digitalocean.com/community/tutorials/how-to-install-and-use-postgresql-on-ubuntu-16-04
- http://fosshelp.blogspot.com/2014/03/how-to-deploy-flask-application-with.html
- https://www.bogotobogo.com/python/Flask/Python_Flask_HelloWorld_App_with_Apache_WSGI_Ubuntu14.php
