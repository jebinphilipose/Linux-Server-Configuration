# Linux Server Configuration

## Project Overview

> Taking a baseline installation of a Linux server and preparing it to host our web applications, securing our server from a number of attack vectors, installing and configuring a database server, and deploying one of our existing web applications onto it.

The server used is a Google Cloud Compute Engine instance.

---
* **Public IP Address: 35.247.82.229**

* **SSH Port: 2200**

* **Live version of the WebApp:** [Food Catalog](http://35.247.82.229.xip.io)

    > Note: To access the live WebApp with Google Authentication use **http://35.247.82.229.xip.io**

---

## Requirements

Things required for this project:

* A browser
* A stable internet connection
* A terminal for connecting to linux server, preferably [git bash](https://git-scm.com/downloads) if on Windows

## Steps

### 1. Creating a Google Compute Engine instance and connecting to the instance

* Create a linux VM instance, see this [guide](https://cloud.google.com/compute/docs/quickstart-linux#before-you-begin) for details
    > Note: Make sure to select **Allow HTTP traffic** in the **Firewall** section.

* To connect to the instance, click on the **SSH** button present under the **Connect** column in the row of the instance

### 2. Updating the server software

* Run the following commands:

    ```
     $ sudo apt-get update
     $ sudo apt-get upgrade
    ```

### 3. Changing the default SSH port | Disable login for `root` user

* Configuring the server instance
    
    1. Run the following command:

       ```
       $ sudo nano /etc/ssh/sshd_config
       ```
    2. Change the line `Port 22` to `Port 2200`
    3. Change the line `PermitRootLogin prohibit-password` to `PermitRootLogin no`
    4. Save and exit the file
    5. Run the following command:

       ```
       $ sudo service sshd restart
       ```

* Configuring the Google Compute Engine firewall
    
    1. Open the [Firewall rules](https://console.cloud.google.com/networking/firewalls/list) page
    2. Click on **CREATE FIREWALL RULE** option
    3. Add the following details:
    
        * Name - `new-allow-ssh`
        * Network - `default`
        * Priority - `65534`
        * Direction of traffic - `Ingress`
        * Allow on match - `Allow`
        * Targets - `All instances in the network`
        * Source IP ranges - `0.0.0.0/0`
        * Protocols and ports - `tcp:2200`

    4. Click on **Create**
    5. Delete the `default-allow-ssh` rule
    6. Go to [VM instances](https://console.cloud.google.com/compute/instances) page and click on the dropdown button under the **Connect** column in the row of the instance and then select **Open in browser window on custom port** option and open on port **2200**. If everything is configured correctly, then you are logged in and therefore the port is changed successfully. Now you can close the previously opened instance window

### 4. Configuring the Uncomplicated Firewall (UFW)

* Configure the firewall to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123)
* Run the following commands:

    ```
     $ sudo ufw default deny incoming
     $ sudo ufw default allow outgoing
     $ sudo ufw allow 2200/tcp
     $ sudo ufw allow www
     $ sudo ufw allow ntp
     $ sudo ufw enable

    ```

### 5. Creating a new user *grader* and giving `sudo` access

* To add the user **grader**, run:
    
    ```
    $ sudo adduser grader
    ```

* To give `sudo` access to **grader**, run:

    ```
    $ sudo nano /etc/sudoers.d/grader
    ```
    > Now in this file, write and save: `grader ALL=(ALL) NOPASSWD: ALL`

### 6. Creating an SSH key pair for **grader** and installing the public key

* On your local machine, create an SSH key pair by running `$ ssh-keygen -C grader` in the terminal. Enter the location where you would save the key. Enter nothing if you dont want a passphrase
* Now back into the linux server instance, login as the **grader** user by running: `$ sudo su - grader`
* Make a new directory: `$ mkdir .ssh`
* Create and open a new file: `sudo nano .ssh/authorized_keys`
* Copy the contents of the public key file(.pub) which you already generated on the local machine with the help of `ssh-keygen` command
* Paste the contents into the `authorized_keys` file, save and exit the file
* Change the permisions of the `.ssh` directory and the `authorized_keys` file:

    ```
    $ chmod 700 .ssh
    $ sudo chmod 644 .ssh/authorized_keys
    ```

* Go to [VM instances](https://console.cloud.google.com/compute/instances) and  click on the name of your instance. In the VM instance details page, click on **EDIT** option. Scroll down the page until you find **SSH Keys** section. Click on **Show and edit** option, then click on **+ Add item** button. Paste your public key file(.pub) contents in the box and then click on the **Save** button  

### 7. Logging in as *grader* user from the local machine

* Run the command:

    ```
    $ ssh -i <full_path_of_pub_file> grader@35.247.82.229 -p 2200
    ```

* Enter the passphrase for the key
* Now you have got access to the linux server instance from your local machine

### 8. Configure the local timezone to UTC

* Check whether your timezone is configured to UTC by running `$ date` in your terminal.
* If the timezone is not set to UTC, run the following command:

    ```
    $ sudo timedatectl set-timezone UTC
    ```

### 9. Install Apache, mod_wgi and Git

* Run the following command:

    ```
    $ sudo apt-get install apache2 libapache2-mod-wsgi git
    ```

* Enable **mod_wsgi**: 

    ```
    $ sudo apache2ctl restart
    ```

* Configure `git`:

    ```
    $ git config --global user.name "<Your-Full-Name>"
    $ git config --global user.email "<your-email-address>"
    ```

### 10. Install and configure PostgreSQL

* Install Python dependencies for Postgresql:

    ```
    $ sudo apt-get install libpq-dev python-dev
    ```
* Install Postgresql:

    ```
    $ sudo apt-get install postgresql postgresql-contrib
    ```
* Check if no remote connections are allowed:

    ```
    $ sudo cat /etc/postgresql/9.5/main/pg_hba.conf
    ```
See this [link](https://www.digitalocean.com/community/tutorials/how-to-secure-postgresql-on-an-ubuntu-vps#do-not-allow-remote-connections) for more details
* Login as *postgres* user (Default user), and get into `psql` shell:

    ```
    $ sudo su - postgres
    $ psql
    ```
* Create a new user named **catalog**: `# CREATE USER catalog WITH PASSWORD 'catalogpass';`
* Create a new database named *catalog*: `# CREATE DATABASE catalog WITH OWNER catalog;`
* Connect to the *catalog* database: `\c catalog`
* Revoke all rights: `# REVOKE ALL ON SCHEMA public FROM public;`
* Grant all permissions to **catalog** user: `# GRANT ALL ON SCHEMA public TO catalog;`
* Get out of `psql` shell: `\q`
* Switch back to **grader** user: `exit`
* Edit the `pg_hba.conf` file by running `$ sudo nano /etc/postgresql/9.5/main/pg_hba.conf`, scroll to almost end of the file and add the following line under **# Database administrative login by Unix domain socket** section:

    ```
    local   all             catalog                                 password
    ```
* Restart PostgreSQL server:

    ```
    $ sudo service postgresql restart
    ```

### 11. Install Flask and other dependencies

* Run the following commands:

    ```
    $ sudo apt-get install python-pip
    $ sudo pip install flask
    $ sudo pip install httplib2 oauth2client sqlalchemy psycopg2 requests
    ```

### 12. Clone the Food Catalog app from GitHub

* Make a directory named *catalog* in **/var/www**:

    ```
    $ sudo mkdir /var/www/catalog
    ```
* Clone the project **Food-Catalog** to the *catalog* directory:

    ```
    $ sudo git clone https://github.com/jebinphilipose/Food-Catalog.git /var/www/catalog
    ```

* Make a *catalog.wsgi* file to handle requests with **mod_wsgi**:

    ```
    $ cd /var/www/catalog
    $ sudo nano catalog.wsgi
    ```

* Enter the following contents, save and exit the file:

    ```
    import sys
    import logging
    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog/")

    from catalog import app as application
    application.secret_key = 'super_secret_key'
    ```

* Change the engine inside the **.py** files:

    ```
    engine = create_engine('postgresql://catalog:catalogpass@localhost/catalog')
    ```

* In `catalog.py` file:

    1. Add the following line in the beginning after the line `app = Flask(__name__)`:

        ```
        app_path = '/var/www/catalog/'
        ```
    2. Change all occurences of line `'client_secrets.json'` to `app_path + 'client_secrets.json'`

* Setup the initial data:

    ```
    $ python database_setup.py
    $ python populate_database.py
    ```

### 13. Configure Apache by editing VirtualHost file

* To edit the file run:

    ```
    $ sudo nano /etc/apache2/sites-available/000-default.conf
    ```
* Edit it to match the following contents:

    ```
    <VirtualHost *:80>
      ServerName 35.247.82.229
      ServerAdmin jebinphilip24@gmail.com
      WSGIScriptAlias / /var/www/catalog/catalog.wsgi
      <Directory /var/www/catalog/>
          Order allow,deny
          Allow from all
      </Directory>
      Alias /static /var/www/catalog/static
      <Directory /var/www/catalog/static/>
          Order allow,deny
          Allow from all
      </Directory>
      ErrorLog ${APACHE_LOG_DIR}/error.log
      LogLevel warn
      CustomLog ${APACHE_LOG_DIR}/access.log combined
    </VirtualHost>
    ```

### 14. Restart Apache to launch the app

* Run the following command:

    ```
    $ sudo apache2ctl restart
    ```

* Access the WebApp in your browser: [Food Catalog](http://35.247.82.229.xip.io)

### 15. Automatically manage package updates

* Install and configure `unattended-upgrades` to automatically install updated packages:

    ```
    $ sudo apt-get install unattended-upgrades
    $ sudo dpkg-reconfigure unattended-upgrades
    ```

## References

* Stack Overflow
* GCP Documentation
* DigitalOcean Community