# Udacity FSWD ND - Linux Server Configuration Project


Host Name: ec2-13-125-243-106.ap-northeast-2.compute.amazonaws.com

IP Address: 13.125.243.106

SSH port : 2200

The complete URL to your hosted web application: http://ec2-13-125-243-106.ap-northeast-2.compute.amazonaws.com

## Requirements

-  python
-  apache
-  git
-  httplib2
-  flask
-  sqlalchemy
-  psycopg2
-  requests
-  oauth2client

## Amazon Lightsail Server Set Up

1. Link to Amazon Lightsail: https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Flightsail.aws.amazon.com%2Fls%2Fwebapp%3Fstate%3DhashArgs%2523%26isauthcode%3Dtrue&client_id=arn%3Aaws%3Aiam%3A%3A015428540659%3Auser%2Fparksidewebapp&forceMobileApp=0

2. Create a new AWS account and then go back to use the link above to log in

3. After you log in, click 'Create Instance'

4. Select Platform and  blueprint

5. Scroll down to name your instance and click 'Create'

6. The instance needs 5~10 mins to set up.

7. Click the status card and you will get into this page. Click the `Account page` at the bottom

8. Click the 'Download' to download your private key, it should go to your Download folder by default. 

9. Click the 'Networking' tab and find the 'Add another' at the bottom. Add port 123 and 2200. Amazon Lightsail allows only port 22 and 80 by default, no matter how you set it up in ubuntu's ufw

## Server Configuration


1. Move your downloaded `.pem` public key file into .ssh folder that is at the root of Finder

2. We need to make our public key usable and secure. 
`$ chmod 600 ~/.ssh/YourAWSKey.pem`

3. We now use this key to log into our Amazon Lightsail Server: `$ ssh -i ~/.ssh/YourAWSKey.pem ubuntu@13.125.243.106`

4. Type  `$ sudo adduser grader` to create another user 'grader'

5. Now create a new file under the sudoers directory: `$ sudo nano /etc/sudoers.d/grader`. Fill that file with `grader ALL=(ALL:ALL) ALL`

6. Run the following commands to update all packages and install finger package:
- `$ sudo apt-get update`
- `$ sudo apt-get upgrade`
- `$ sudo apt-get install finger`

7. Open a new Terminal window and input `$ ssh-keygen -f ~/.ssh/udacity_key.rsa`

8.  `$ cat ~/.ssh/udacity_key.rsa.pub` to read the public key. Copy the public key.

9. Create a .ssh directory in grader : `$ mkdir .ssh`

10. Create a file to store the public key: `$ touch .ssh/authorized_keys`

11. Paste the authorized_keys file `$ nano .ssh/authorized_keys`

12. Change the permission: `$ sudo chmod 700 /home/grader/.ssh` and `$ sudo chmod 644 /home/grader/.ssh/authorized_keys`

13. Change the owner from root to grader: `$ sudo chown -R grader:grader /home/grader/.ssh`

14. Restart the ssh service: `$ sudo service ssh restart`

15. Exit and Log into the server as grader: `$ ssh -i ~/.ssh/udacity_key.rsa grader@13.125.243.106`

16. We now need to enforce the key-based authentication: `$ sudo nano /etc/ssh/sshd_config`. Find the *PasswordAuthentication* line and change text after to `no`. After this, restart ssh again: `$ sudo service ssh restart`

17. We now need to change the ssh port from 22 to 2200, as required by Udacity: `$ sudo nano /etc/ssh/ssdh_config` Find the *Port* line and change `22` to `2200`. Restart ssh: `$ sudo service ssh restart`

18. Disconnect the server by `$ ~.` and then log back through port 2200: `$ ssh -i ~/.ssh/udacity_key.rsa -p 2200 grader@13.125.243.106`

19. Disable ssh login for *root* user, as required by Udacity: `$ sudo nano /etc/ssh/sshd_config`. Find the *PermitRootLogin* line and edit to `no`. Restart ssh `$ sudo service ssh restart`

20. Now we need to configure UFW to fulfill the requirement:
- `$ sudo ufw allow 2200/tcp`
- `$ sudo ufw allow 80/tcp`
- `$ sudo ufw allow 123/udp`
- `$ sudo ufw enable`


## Deploy Catalog Application

We will use a virtual machine, apache2, and postgre to host our application. 

1. Install required packages
- `$ sudo apt-get install apache2`
- `$ sudo apt-get install libapache2-mod-wsgi python-dev`
- `$ sudo apt-get install git`

2. Enable mod_wsgi by `$ sudo a2enmod wsgi` and start the web server by `$ sudo service apache2 start` or `$ sudo service apache2 restart`.

3. Set up the folder structure (although clumsy, but works)
- `$ cd /var/www`
- `$ sudo mkdir catalog`
- `$ sudo chown -R grader:grader catalog`
- `$ cd catalog`

4. Now we clone the project from Github: `$ git clone [your link] catalog` 

5. Create a .wsgi file: `$sudo nano catalog.wsgi` and add the following into this file
```
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'supersecretkey'
```

6. Rename the `application.py` to `__init__.py`

7. Now we need to install and start the virtual machine
- `$ sudo pip install virtualenv`
- `$ sudo virtualenv venv`
- `$ source venv/bin/activate`
- `$ sudo chmod -R 777 venv`

8. Now we need to install the Flask and other packages needed for this application
- `$ sudo apt-get install python-pip`
- `$ sudo pip install Flask`
- `$ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlaclemy_utils #etc `

9. Use the `nano __init__.py` command to change the `client_secrets.json` line to `/var/www/catalog/catalog/client_secrets.json` 
and change the host to your Amazon Lightsail public IP address and port to 80

10. Now we need to configure and enable the virtual host
- `$ sudo nano /etc/apache2/sites-available/catalog.conf`
- Paste the following code and save
```
<VirtualHost *:80>
    ServerName [YOUR PUBLIC IP ADDRESS]
    ServerAlias [YOUR AMAZON LIGHTSAIL HOST NAME]
    ServerAdmin admin@35.167.27.204
    WSGIDaemonProcess catalog python-path=/var/www/catalog:/var/www/catalog/venv/lib/python2.7/site-packages
    WSGIProcessGroup catalog
    WSGIScriptAlias / /var/www/catalog/catalog.wsgi
    <Directory /var/www/catalog/catalog/>
        Order allow,deny
        Allow from all
    </Directory>
    Alias /static /var/www/catalog/catalog/static
    <Directory /var/www/catalog/catalog/static/>
        Order allow,deny
        Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

11. Now we need to set up the database
- `$ sudo apt-get install libpq-dev python-dev`
- `$ sudo apt-get install postgresql postgresql-contrib`
- `$ sudo su - postgres -i`

You should see the username changed again in command line, and type `$ psql` to get into postgres command line

12. Now we create a user to create and set up the database. I name my database `catalog` with user `catalog`
- `$ CREATE USER catalog WITH PASSWORD [your password];`
- `$ ALTER USER catalog CREATEDB;`
- `$ CREATE DATABASE catalog WITH OWNER catalog;`
- Connect to database `$ \c catalog`
- `$ REVOKE ALL ON SCHEMA public FROM public;`
- `$ GRANT ALL ON SCHEMA public TO catalog;`
- Quit the postgrel command line: `$ \c` and then `$ exit`

13. use `sudo nano` command to change all engine to `engine = create_engine('postgresql://catalog:[your password]@localhost/catalog`

14. Initiate the database if you have a script to do so:

15. Restart Apache server `$ sudo service apache2 restart` and enter your public IP address or host name into the browser. 

## Reference

- https://github.com/rrjoson/udacity-linux-server-configuration/blob/master/README.md

- https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps

- https://github.com/kongling893/Linux-Server-Configuration-UDACITY

