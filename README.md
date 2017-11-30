# Linux-Server Configuration for mocCatalogue using Amazon Lightsail

## Details
IP Address: 18.217.114.163  
SSH: Port 2200  
URL: http://ec2-18-217-114-163.us-east-2.compute.amazonaws.com  
Third Part Resources: None  

## Setup Instructions

### Creating Amazon Lightsail Instance 
You will first need to sign-up for Amazon Lightsail and create and instance. You can sign up [here](https://amazonlightsail.com)  
* Click "Create Instance"  
* Select "OS Only", then "Ubuntu"  
* Click "Create"   

You now have an instance of a linux server, we will now need to do some configurations on lightsail:  
### Change the SSH port from 22 to 2200  
* Select your instance  
* Go to "Networking" tab.  
* Under "Firewall", add a custom application of protocal TCP on port 2200. Select the "Connect" tab.  
* Click "Connect using SSH".  
* In the terminal, update the system package using: 
```
sudo apt-get update 
sudo apt-get upgrade
```
* We will now make some configurations to the ```ssh_config``` file. Type in the command:
```sudo nano /etc/ssh/sshd_config```  
The port number should say 20 which is the default port, change this to 2200.
* Restart the ssh service using: 
```sudo service sshd restart```
* Run the following commands in the terminal to configure the firewall:
```
sudo ufw default deny incoming  
sudo ufw default allow outgoing
sudo ufw allow 2200/tcp  
sudo ufw allow www  
sudo ufw allow ssh  
sudo ufw allow ntp  
sudo ufw enable
```  

* Set the timezone to UTC by running the following command: 
```sudo timedatectl set-timezone UTC```

### Create a new key-pair using ssh-keygen  
* On your local machine in git bash, run the command
```ssh-keygen```  
* Choose a location to save the file and enter a passphrase if you would like. 
Make a note of where the files are located we will need them later
* Login to your server using ssh from git bash on your local machine.
* In the instance in AWS Lighsail, click "Account Page" at the bottom of the "Connect" tab.
* Under "SSH Keys", download the private key.  
* Move the downloaded private key to 
```C:/Users/[YOUR USERNAME]/.ssh```   
* Open a git terminal and log in to the server with the following command: 
```ssh -i ~/.ssh/[Amazonian.pem] -p 2200 ubuntu@[your instance's public IP address]```

### Create a new user named "grader" with sudo permissions  
* Create a user named "grader" by using the following command: 
```sudo adduser grader```
* Create a new file named "grader" in the sudoers directory, and open the new file in Nano: 
```sudo nano /etc/sudoers.d/grader```  
* Add the following text to ```/etc/sudoers.d/grader```: 
```grader ALL=(ALL:ALL) ALL```  
* On your local machine, locate the rsa file you generated with ssh-keygen. Copy the contents of this file
* On the server terminal run
```sudo nano /.ssh/authorized_keys```   
* Paste the contents of the rsa file, this key can now be used to log into the server  
* Change file permissions by entering the following commands: 
```
chmod 700 .ssh
chmod 644 .ssh/authorized_keys
```
* On your local machine, log into the server by running the command
```ssh grader@18.217.114.163 -p 2200```

### Changing ```sshd_config```  
We will now make some change to ```sshd_config```. 
* Run ```sudo nano /etc/.ssh/sshd_config``` 
* Change the line PermitRootLogin prohibit-password to ```PermitRootLogin no``` 
* Add: ```AllowUser grader``` so grader can still remotely log in
* Change ```PasswordAuthentication``` to ```PasswordAuthentication no```  
* Run ```sudo service ssh restart```
You can now only log in to "grader" using the command  
```ssh -i your_generated_rsa grader@18.217.114.163 -p 2200```

### Installing necessary programs
For the following steps to work, you will need these programs:
```
wsgi
linux-aws
apache2
libapache2-mod-wsgi
postgresql
python-pip
libpq-dev
SQLAlchemy
Flask
psycopg2
oauth2client
httplib2
requests
```
You can download these using '''sudo apt-get''' or '''pip install''' once pip has been installed

### Edit .conf
Now, let's change  the ```.conf``` file. 
* Open up the file in nano using: ```sudo nano /etc/apache2/sites-enabled/000-default.conf```  
We could add our own ```.conf``` file, but since we are only hosting one application there is no need. 
* Edit the file so it matches the following:  
```
<VirtualHost *:80>

        ServerAdmin you@example.com
        ServerName yourApp.com
        ServerAlias ec2-XX-XXX-XXX-XXX.us-east-2.compute.amazonaws.com (Replace Xs with your server's public IP)

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

    WSGIDaemonProcess yourAppName user=ubuntu
    WSGIScriptAlias / /var/www/yourAppName/yourAppName.wsgi
    <Directory /var/www/yourAppName>
        WSGIProcessGroup yourAppName
        WSGIApplicationGroup %{GLOBAL}
        Order deny,allow
        Allow from all
    </Directory>
</VirtualHost>
```
Note that you should change the directories to match the directories on your server.  

### Clone github repo
* Navigate to the directory ```/var/www``` and clone your project's repo. 

### Make changes to files
There are a couple changes we need to make to our files in order for them to work properly with the server. 
* In the appropriate files change all create_engine statements to: ```create_engine(postgresql://catalog:catalog@localhost/catalogitems)```. 
* You also need to change any references to files to the new directory on the server.

### Create new wsgi file
Create a new .wsgi with the same title as your app and make it the following: 
```
import sys
import logging

logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, '/var/www/mocCatalogue')

from views import app as application

application.secret_key = 'super_secret_key'
```


### Install and Configure postgresql

* Install postgresql using ```sudo apt-get postgresql```
* Once installed, log in using ```sudo -u postgres -i```. Type ```psql``` to enter psql
* Create a new user "catalog": ```CREATE USER catalog WITH PASSWORD 'catalog';```
* Create a new database: ```CREATE DATABASE catalogitems WITH OWNER catalog;```
* Quit psql ```\q;```

### Change client_secrets

You will need to change the permissions for your app in Google Developers. 
* Go to the [Google Developers API Console](console.developers.google.com/apis) 
and navigate to your OAuth 2.0 client ID under credentials. 
* Add your URL (ec2-XX-XXX-XXX-XXX.us-east-2.compute.amazonaws.com) to "Authorized
Javascript Origins" and "Authorized redirect URIs". 
* Download the JSON file and save the settings. 
* Now go to the client_secrets file on your server and
replace it with the client_secrets file you just downloaded. 

### Setup app

* Navigate to your project files. 
* Populate your database using ```python yourDatabasePopulationFile.py```. If there are no errors you have done all steps correctly. 
If not, make sure you changed the engine and directories correctly, and downloaded the necessary software. 

### Launch Website

Now for the moment of truth, in your webrowser enter in the URL for your website. If there's a server error, you can check the apache error logs at:
```/var/logs/apache2/error.log```
