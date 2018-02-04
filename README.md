# Udacity Linux Configuration Note

-----------------------------------------

## Preface

This is the Udacity FSND final project about Linux server configuration on [AWS Lightsail](https://lightsail.aws.amazon.com).

## Server

The web server address is following:

* IP address: [18.216.73.137](http://18.216.73.137/)
* Accessible ssh port: 2200

And you can login the server with `ssh grader@18.216.73.137 -p 2200` and the password is `Udcourse#0812`. Or you also can login with `ssh -i /home/user/.ssh/udacity_key.rsa grader@18.216.73.137 -p 2200`, and the rsa key is [here](https://github.com/eryue0220/linux-configuration-note/blob/master/udacity_key.rsa).

## Configuration

And now we will talk about the server configuration in details

### Configuration UFW

According to the required, we just only allowing the connections for SSH (port 2200), HTTP(port 22) and NTP(port 123), and we can use the command as follow:

* `sudo ufw allow 2200/tcp`
* `sudo ufw allow www`
* `sudo ufw allow 123/udp`
* `sudo ufw enable`

And now we start the UFW, and we can use the command `sudo ufw status` to see the current ufw is active or inactive

### Create New user

Now we would add a new user named `grader`, running the command in the terminal:

```bash
sudo adduser grader
```

and then following the tips on the screen, such as setting a new password, a full name and so on.

After that, we also need to give it to the sudo permission. And just one easy command:

```bash
sudo adduser grader sudo
```

Now the user `grader` has the sudo permission.

### Change SSH port

Now, we would change the ssh port from `22` to `2200`. change the directory to `/etc/ssh/`. It is a good behavior to backup the file before we modified it. Running the code to backup:

```bash
sudo cp sshd_config sshd_config_backup
```

and then we can modified the `sshd_config` and change the line `Port 22` to `Port 2200`, also we also need to change the line `PermitRootLogin without-password` to `PermitRootLogin no`. Final we also change the `PubkeyAuthentication yes` to `PubkeyAuthentication no`.

Now in your local machine run the command to produce a key for the sso connection. Running the code:

```bash
ssh-keygen -f ~/.ssh/udcourse.rsa
```

And we will see the tow file: `udcourse.rsa` and `udcourse.rsa.pub`. And we paste the content in `udcourse.rsa.pub` into the `/home/user/.ssh/authorized_keys` on your remote machine. Restart the ssh service let the modifications take effect, so run the commaind:

```bash
sudo service ssh restart
```

Now we can login the server through the command:

```bash
ssh -i ~/.ssh/udcourse.rsa -p 2200 grader@34.205.69.230
```

### Change Timezon

If the date is not UTC and we can change it through:

```bash
sudo timedatectl set-timezone UTC
```

### Update the system software

Make sure our system installed package is fresh. we can use these two command:

* Download the package lists by `sudo apt-get update`
* Fetch new versios of packages by `sudo apt-get upgrade`

### Install lists of softwares

#### Apache

In order to visit our catalog website online. We need a web server, we choice [apache](http://www.apache.org/) for convince. Install apache following the command:

```bash
sudo apt-get install apache2
```

#### mod_wsgi

* Run `sudo apt-get install libapache2-mod-wsgi`
* Start the web server through `sudo service apache2 start`

#### Git

Install git with `sudo apt-get install git`. Then create a directory `catalog` in `/var/www/`, and clone the project `https://github.com/eryue0220/item-catalog` to it by:

```bash
git clone https://github.com/eryue0220/item-catalog ./catalog
```

Now we create a file call `catalog.wsgi`, and paste the followint content to it:

```python
import sys
import logging
logging.basicConfig(stream=sys.stderr)
sys.path.insert(0, "/var/www/catalog/")

from catalog import app as application
application.secret_key = 'super_secret_key'
```

#### Install Flask and other dependencies

Before install the flask framework, we need install the `python-pip` first. We can do it with `sudo apt-get install python-pip`. When finished, we can install Flask and other dependencies through:

```bash
pip install flask sqlalchemy requests httplib2 oauth2client sqlalchemy_utils psycopg2
```

### Configuration the apache2

We create a new conf name `catalog.conf` for our website in `/etc/apache2/sites-avilable/` by `sudo vim catalog.conf`. Then paste the following code into it:

```apache
<VirtualHost *:80>
    ServerName 18.216.73.137
    ServerAdmin admin@18.216.73.137
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

Before we start our web server, we should stop the default server by `sudo a2dissite 000-default.conf`. Maybe occur some warn when do that, you can reload and restart your apache server through `sudo service apache2 reload` and `sudo service apache2 restart`, after that running the code `sudo a2dissite 000-default.conf` again.

Now we can start our service by `sudo a2ensite catalog.conf`. Then restart the apache service with `sudo service apache2 restart` to take effect.

### Install and Configure Postgresql

First we need to install postresql.

```bash
sudo apt-get install libpq-dev postgresql postgresql-contrib
```

Then we start configuring the psql to use in our web production. And just type the following command in the terminal.

* `sudo su - postgres`
* `psql`
* `CREATE USER catalog WITH PASSWORD 'password';`
* `ALTER USER catalog CREATEDB;`
* `CREATE DATABASE catalog WITH OWNER catalog;`
* `\c catalog`
* `REVOKE ALL ON SCHEMA public FROM public;`
* `GRANT ALL ON SCHEMA public TO catalog;`
* `\q`
* `exit`

### Update the project

Now we need do some change for the project:

* Modified the `client_secrets.json` to `/var/www/catalog/catalog/client_secrets.json` in your `application.py` in `/var/www/catalog/catalog/`
    * change the code about create engine line in the `db_session` and `db/db_set.py` to `engine = create_engine('postgresql://catalog:password@localhost/catalog')`
* change the `application.py` to `__init__.py`

### Final

Now, all is done. But before we visit the result, we may restart or our apache2 server to make sure our change has taken effect. Running the code `sudo service apache2 restart`.

If no accident, now we can visit the website online 👏👏.

> Tips
> If you open the browser, but see the error page such as `The Internal Error` and so on, you can checkout the log through `sudo less /var/log/apache2/error.log` to help you debug.