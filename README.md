# Linux-server-configurations

IP address - http://13.234.150.58
App URL - http://13.234.150.58.xip.io

###### Step 1 - Start a Ubuntu Linux Server instance on Amazon Lightsail
- Login into your AWS account which has Lightsail access.
- Once you have logged in, click on `create instance`
- Choose `Linux/Unix` platform ,`OS only` and select `Ubuntu 16.04`
- Choose the cheapest instance plan and name your instance.
- Click the `Create` button to create the instance.
- Wait for instance to startup

###### Step 2 - SSH into the server
- From the `Account` menu on Lightsail click on `SSH Keys` and download the default private key.
- Move the private key file into your local folder `~/.ssh` and rename it as `lightsail_key.rsa`
- In your terminal type `chmod 600 ~/.ssh/lightsail_key.rsa.pem`
- To connect to the instance via your terminal `ssh -i ~/.ssh/lightsail_key.rsa.pem ubuntu @13.234.150.58`

###### Step 3 - Update and upgrade packages
 `sudo apt-get update`
 `sudo apt-get upgrade`
 
###### Step 4 - Change port 22 to 2200
 `sudo nano /etc/ssh/sshd_config`. Change port 22 to 2200. Save and restart ssh `sudo service ssh restart`.
 
###### Step 5 - Configure UFW
 ```
 sudo ufw status                  
sudo ufw default deny incoming   
sudo ufw default allow outgoing  
sudo ufw allow 2200/tcp          
sudo ufw allow www               
sudo ufw allow 123/udp           
sudo ufw deny 22                 
```
Enable UFW `sudo enable ufw`
Exit SSH connection `exit`

Click on `Manage` option in your lightsail instance. In the `Networking` tab change firewal settings as per required.

###### Step 6 - Create new user grader
  `sudo adduser grader`
  Fill the information. Enter password `grader` for the user
  
 ###### Step 7 - Give grader user sudo access
 Edit sudoers file `sudo visudo`
 Add line `grader ALL=(ALL:ALL) ALL`
 Save and exit
 
 ###### Step 8 - Create an SSH-Keypair for grader using ssh-keygen tool
 On the local machine run `ssh-keygen`
 Save the key in local directory `~/.ssh`
 Enter a passphrase for the key
 2 files will be generated. Copy the contents of the `.pub` file.
 Login to grader VM
 - Create a directory `mkdir .ssh`
 - Run `sudo nano /.ssh/authorized_keys` and paste the contents into this file. Save and exit.
 - Give permissions `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
 - Check `/etc/ssh/sshd_config` file if `PasswordAuthentication` is set to no.
 - Restart SSH `sudo service ssh restart`
 On the local machine run `ssh -i ~/.ssh/your_key_file -p 2200 grader@13.234.150.58`
 
 ###### Step 8 - Configure timezone to UTC
 While logged in as `grader` type `sudo dpkg-reconfigure tzdata` 
 Click `none of the above` and then chose `UTC`.
 
 ###### Step 9 - Install and configure apache to serve wsgi app
 - While logged in as grader `sudo apt-get install apache2`
 - Install python mod_wsgi package `sudo apt-get install libapache2-mod-wsgi`
 - Enable mod_wsgi using `sudo a2enmod wsgi`
 
 ###### Step 10 - Install and configure postgresql
 - Install postgresql using `sudo apt-get install postgresql`
 - Switch to postgres user using `sudo su - postgres`
 - Open PostgreSQL terminal by typing `psql`
 - Create `catalog` user with password and give them authority to create databases,
    ```
    postgres=# CREATE ROLE catalog WITH LOGIN PASSWORD 'catalog';
    postgres=# ALTER ROLE catalog CREATEDB;
    ```
 - Switch back to user `grader`
 - Create new user `catalog` using `sudo adduser catalog`. Enter password as `catalog`.
 - Give sudo access to user `catalog`
 - Run `sudo visudo` and add line `catalog ALL=(ALL:ALL) ALL`. Save and exit.
 - While logged in as `catalog` create database `createdb catalog`
 - Switch back to user `grader`
 
 ###### Step 11 - Install git 
 `sudo apt-get install git`
 
 ###### Step 12 - Clone Item Catalog project form github
 - While logged in as `grader` create directory `/var/www/Catalog`
 - Change to that directory using `cd` command and clone project using `git clone https://github.com/VinayPoduval/Item-Catalog.git Catalog`
 - Change ownership of Catalog directory to `grader` using `sudo chown -R grader:grader Catalog/`
 - Rename `itemcatalog.py` to `__init__.py`
 - In `__init__.py` replace 
    ```
    # app.run(host="0.0.0.0", port=5000, debug=True)
    app.run()
    ```
 - In `database_setup.py` replace 
    ```
    # engine = create_engine("sqlite:///catalog.db")
    engine = create_engine('postgresql://catalog:catalog@localhost/catalog')
    ```
    
###### Step 13 - Install virtual environment and dependencies
- While logged in as `grader` install `sudo apt-get install python-pip`
- Install virtual environment `sudo apt-get install python-virtualenv`
- Change to `/var/www/Catalog/Catalog` directory 
- Create virtual environment `sudo virtualenv -p python venv`
- Change ownership to `grader` with `sudo chown -R grader:grader venv/`
- Activate the new environment `. venv/bin/activate`
- Install the dependencies 
   ```
   pip install httplib2
  pip install requests
  pip install --upgrade oauth2client
  pip install sqlalchemy
  pip install flask
  sudo apt-get install libpq-dev
  pip install psycopg2
  ```
- Run `python __init__.py` 
- Deactivate virtual environment using `deactivate`

###### Step 14 - Set up and enable virtual host
- Add following line in `/etc/apache2/mods-enabled/wsgi.conf` to use Python
  ```
  #WSGIPythonPath directory|directory-1:directory-2:...
  WSGIPythonPath /var/www/Catalog/Catalog/venv/lib/python2.7/site-packages
  ```
- Create `/etc/apache2/sites-available/catalog.conf` file and add the following lines in it to configure virtual host
  ```
  <VirtualHost *:80>
    ServerName 13.234.150.58
  ServerAlias 13.234.150.58.xip.io
    WSGIScriptAlias / /var/www/Catalog/catalog.wsgi
    <Directory /var/www/Catalog/Catalog/>
    	Order allow,deny
  	  Allow from all
    </Directory>
    Alias /static /var/www/Catalog/Catalog/static
    <Directory /var/www/Catalog/Catalog/static/>
  	  Order allow,deny
  	  Allow from all
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/error.log
    LogLevel warn
    CustomLog ${APACHE_LOG_DIR}/access.log combined
  </VirtualHost>
  ```
- Enable virtual host using `sudo a2ensite catalog`
- Reload apache service using `sudo service apache2 reload`

###### Step 15 - Set up Flask app
- Create `/var/www/Catalog/catalog.wsgi` file and add the following lines
  ```
  activate_this = '/var/www/Catalog/Catalog/venv/bin/activate_this.py'
  with open(activate_this) as file_:
    exec(file_.read(), dict(__file__=activate_this))

  #!/usr/bin/python
  import sys
  import logging
  logging.basicConfig(stream=sys.stderr)
  sys.path.insert(0, "/var/www/Catalog/Catalog/")
  sys.path.insert(1, "/var/www/Catalog/")

  from catalog import app as application
  application.secret_key = "..."
  ```
- Restart apache service using `sudo service apache2 restart`

###### Step 16 - Set up database schema 
- Edit `/var/www/Catalog/Catalog/catalog_database_setup.py`
- Add these 2 lines in the beginning 
  ```
  import sys
  sys.path.insert(0, "/var/www/Catalog/Catalog/venv/lib/python2.7/site-packages")
  ```
- From the `/var/www/Catalog/Catalog/` directory activate virtual environment using `. venv/bin/activate`
- Run `python catalog_database_setup.py`
- Deactivate virtual environment `deactivate`

###### Step 17 - Disable default Apache site
- Run `sudo a2dissite 000-default.conf` 
- Reload apache service using `sudo service apache2 reload`

###### Step 18 - Launch the web app
- Change ownership of project directories `sudo chown -R www-data:www-data Catalog/`
- Restart apache service `sudo service apache2 restart`
- Open browser and open http://13.234.150.58

## References
- Digital Ocean [How to deploy Flask apps on Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-    application-on-an-ubuntu-vps)
- Flask documentation [Virtual Environments](http://flask.pocoo.org/docs/0.12/deploying/mod_wsgi/#working-with-virtual-environments)
- Digital Ocean [Secure PostgreSQL on Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps)
- Digital Ocean [Setup SSH keys](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys--2)
- Official Ubuntu Documentation [UFW - Uncomplicated Firewalls](https://help.ubuntu.com/community/UFW)
- Server Pilot [Create servers using Amazon Lightsail](https://serverpilot.io/docs/how-to-create-a-server-on-amazon-lightsail)
- Github repo [Linux server config](https://github.com/ManishPoduval/LinuxServerConfigurations)
