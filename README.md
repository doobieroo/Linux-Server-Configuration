# Linux Server Configuration    
This project is to configure a Linux virtual machine in order to support the [Item Catalog project](https://github.com/doobieroo/Item-Catalog).

See [http://54.191.211.165](http://54.191.211.165) to see the project live.

The notes are the steps required to get this project set up and running.
Task 1 - Get Your Server
Task 2 - Give Grader Access
Task 3 - Secure Your Server
Task 4 - Prepare to Deploy Your Project
Task 5 - Deploy the Item Catalog Project

## Task 1 - Get Your Server
1. Start a new Ubuntu Linux server instance on [Amazon Lightsail](https://amazonlightsail.com/?sc_channel=PS&sc_campaign=pac_ps_q4&sc_publisher=google&sc_medium=lightsail_b_pac_q42017&sc_content=lightsail_e&sc_detail=amazon%20lightsail&sc_category=lightsail&sc_segment=229758835864&sc_matchtype=e&sc_country=US&sc_geo=namer&sc_outcome=pac&s_kwcid=AL!4422!3!229758835864!e!!g!!amazon%20lightsail)

2. SSH into your new server.
    + Download the default private key (make sure to also write down your public IP address)
    + Move the default private key file to the folder ~/.ssh/
    `mv ~/WHERE_YOU_DOWNLOADED_FILE/private_key.pem ~/.ssh/`
    + Set file rights
    `chmod 600 ~/.ssh/private_key.pem`
    + SSH into the new server instance from your local machine
    `ssh -i ~/.ssh/private_key.pem ubuntu@PUBLIC_IP_ADDRESS`

    Note: for my Lightsail instance - my public IP address is 54.191.211.165

## Task 2 - Give Grader Access
1. Create a new user account named grader.
    + `sudo adduser grader`, enter password of your choosing, fill in fields
    + optional: `sudo apt-get install finger` and then use `finger grader` to verify data

2. Give grader the permission to sudo.
    + `sudo touch /etc/sudoers.d/grader`
    + `sudo nano /etc/sudoers.d/grader`, then type `grader ALL=(ALL:ALL) ALL`, save and quit

3. Create an SSH key pair for grader using the ssh-keygen tool.
    + On your local machine - use `ssh-keygen` to create a private key, then save it in ~/.ssh on the local machine
    
    + On your virtual machine - login as grader
    + `su - grader`
    + `mkdir .ssh`
    + `touch .ssh/authorized_keys`
    + `nano .ssh/authorized_keys`
    + Copy the public key generated on your local machine to this file and save (within Amazon Lightsail you can do this upload via the Network tab on the instance you're using)
    + `chmod 700 .ssh`
    + `chmod 644 .ssh/authorized_keys`
    + Reload SSH service using `sudo service ssh restart`
    + Test this by attempting to login with: `ssh -i PRIVATE_KEY_FILENAME grader@PUBLIC_IP_ADDRESS`

## Task 3 - Secure Your Server
1. Update all currently installed packages.
    + Update the list of currently installed packages
    `sudo apt-get update`
    + Upgrade the currently installed packages
    `sudo apt-get upgrade`

2. Change the SSH port from 22 to 2200.
    + Go to your Lightsail instance online and click the "Networking" tab, under the "Firewall" heading, add a 'Custom' TCP port of 2200, save changes
    + Open config file and change SSH port from 22 to 2200
    `sudo nano /etc/ssh/sshd_config`
    Change these lines from:
    `# What ports, IPs and protocols we listen for`
    `Port 22` 
    to:
    `# What ports, IPs and protocols we listen for`
    `Port 2200`

3. Require key based logins and prevent logins as root.
    + Open config file
    `sudo nano /etc/ssh/sshd_config`
    + Change these lines from:
    `PermitRootLogin yes`
    to `PermitRootLogin no`

    + AND
    `PasswordAuthentication yes`
    to `PasswordAuthentication no`

    + Reload ssh service
    `sudo service ssh restart`

4. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).
    + Check the ufw status.
    `sudo ufw status`
    
    + Allow specific connections.
    + `sudo ufw allow 2200/tcp`
    + `sudo ufw allow 80/tcp`
    + `sudo ufw allow 123/udp`
    
    + Turn on firewall.
    `sudo ufw enable`

## Task 4 - Prepare to Deploy Your Project
1. Configure the local timezone to UTC.
    + Run the following command to set the time to UTC
    `sudo dpkg-reconfigure tzdata`, then select "Other", and finally "UTC"

2. Install and configure Apache to serve a Python mod_wsgi application.
    + Install Apache
    `sudo apt-get install apache2`

    + Start Apache
    `sudo service apache2 start` - verify this install by going to your public ip address and looking for the "it works!" default Apache screen
    
    + Install mod_wsgi
    `sudo apt-get install libapache2-mod-wsgi python-dev`
        (Note: if you built your project with Python 3, you will need to install the Python 3 mod_wsgi package on your server: `sudo apt-get install libapache2-mod-wsgi-py3`.
    
    + Enable mod_wsgi
    `sudo a2enmod wsgi`

    + Configure Apache to handle requests using the WSGI module
    `sudo nano /etc/apache2/sites-enabled/000-default.conf`
    
    + Add `WSGIScriptAlias / /var/www/catalog/catalog.wsgi` before `</VirtualHost>` closing line

3. Install a Virtual Environment
    + Install pip
    `sudo apt-get install python-pip`
    
    + Install the Virtual Environment 
    + `sudo pip install virtualenv`
    + `sudo virtualenv venv`
    + `sudo chmod -R 777 venv`
    + `source venv/bin/activate`
    
4. Install Flask (within the virtual environment - you'll see 'venv' before the user in the command line) 
    `sudo pip install Flask`

    + Install other needed libraries (also within the virtual environment)
    `sudo pip install httplib2 oauth2client sqlalchemy psycopg2 sqlalchemy_utils`

5. Install and configure PostgreSQL.
    + `sudo apt-get install libpq-dev python-dev`
    + `sudo apt-get install postgresql postgresql-contrib`

    + Add catalog user and login as default postgres user to create db and grant permissions to catalog user
    + `sudo adduser catalog`
    + `sudo su - postgres`
    + `psql`
    + `CREATE USER catalog WITH PASSWORD 'catalog';`
    + `ALTER USER catalog CREATEDB;`
    + `CREATE DATABASE shelflife WITH OWNER catalog;`
    + `\c catalog`
    + `REVOKE ALL ON SCHEMA public FROM public;`
    + `GRANT ALL ON SCHEMA public TO catalog;`
    + `\q`
    + `exit`
   
## Task 5 - Deploy the Item Catalog Project
1. Install git
    `sudo apt-get install git`

2. Install Item Catalog from github
    + Change to the www directory
    `cd /var/www`

    + Make a new directory called catalog
    `sudo mkdir catalog`

    + Change the owner to grader
    `sudo chown -R grader:grader catalog`
    
    + Change to the new directory
    `cd catalog`

    + Clone the github item-catalog repository with
    `git clone https://github.com/doobieroo/item-catalog`

    + Change the item-catalog directory to item_catalog
    `sudo mv item-catalog item_catalog`

    + Create a new catalog.wsgi file in the /var/www/catalog/ directory and open it in nano
    `sudo nano catalog.wsgi`

    + Add the following code:
    ```
    #!usr/bin/python
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0,"/var/www/catalog")
    from item_catalog import app as application
    application.secret_key = 'super_secret_key
    ```
    Save the changes and exit

    + Copy your main project file (shelflife.py) into the init.py file
    `mv shelflife.py __init__.py`

    + Make the following changes to `shelflife_models.py` and `shelflife_load.py`:
    `sudo nano shelflife_models.py`
    + Change from `engine = create_engine('sqlite:///shelflife.db')` 
    to `engine = create_engine('postgresql://catalog:catalog@localhost/shelflife')`
    `sudo nano shelflife_load.py`
    + Change from `engine = create_engine('sqlite:///shelflife.db')` 
    to `engine = create_engine('postgresql://catalog:catalog@localhost/shelflife')`
 
    + Create tables and populate with initial data
    + `python /var/www/catalog/item_catalog/shelflife_models.py`
    + `python /var/www/catalog/item_catalog/shelflife_load.py`

    + Make the following changes to `__init__.py`:
    `sudo nano shelflife.py`
    + Change from `engine = create_engine('sqlite:///shelflife.db')` 
    to `engine = create_engine('postgresql://catalog:catalog@localhost/shelflife')`
    + Also change every reference from `client_secrets.json` to `/var/www/catalog/item_catalog/client_secrets.json`

4. Configure and enable virtual host
    `sudo nano /etc/apache2/sites-available/caalog.conf`
    and add this code:
    ``` <VirtualHost *:80>
       ServerName YOUR_PUBLIC_IP_ADDRESS
       ServerAdmin admin@YOUR_PUBLIC_IP_ADDRESS
       ServerAlias YOUR_HOST_NAME
       WSGIScriptAlias / /var/www/catalog/catalog.wsgi
       <Directory /var/www/catalog/item_catalog/>
           Order allow,deny
           Allow from all
       </Directory>
       Alias /static /var/www/catalog/item_catalog/static
       <Directory /var/www/catalog/item_catalog/static/>
           Order allow,deny
           Allow from all
       </Directory>
       ErrorLog ${APACHE_LOG_DIR}/error.log
       LogLevel warn
       CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```
    Save file and exit

    + Enable the virtual host
    `sudo a2ensite catalog`

5. Make sure that your .git directory is not publicly accessible via a browser.
    + `cd var/www/catalog/`
    + `sudo nano .htaccess`
    + Add `RedirectMatch 404 /\.git`
    + Save file and exit

6. Alter OAuth to work with hosted app
    + Ensure your host name (can be found using [http://wwww.hcidata.info/host2ip.cgi](http://wwww.hcidata.info/host2ip.cgi)) is in the ServerAlias entry in the catalog.conf file
    + In the Google Developer Console add:
    host name and IP address to Authorized Javascript origins
    HOST NAME/oauth2callback to the Authorized Redirect URIs
    + Restart apache server
    `sudo apache2 restart`


## Credits
- [How to Deploy a Flask Application on an Ubuntu VPS](https://www.digitalocean.com/community/tutorials/how-to-deploy-a-flask-application-on-an-ubuntu-vps#setup)


## License
This project is licensed under the GNU General Public License. See the [LICENSE.md](https://github.com/doobieroo/Linux-Server-Configuration/blob/master/LICENSE) for details.




