# Linux Server Configuration

A project to learn to setup and configure a Linux (Ubuntu) web server. A baseline installation of a Linux server was taken and prepared to host web applications. The server was secured from a number of attack vectors. A database server was installed and configured and one of my existing web applications were deployed onto it.

**Note:** This is a solution to project 6 of the [Udacity Full-Stack Web Developer Nanodegree Program][1] based on the courses [Linux Command Line Basics (ud595)][2] and [Configuring Linux Web Servers (ud299)][3].

## Quick Start

| Name               | Value         |
|--------------------|---------------|
| IP Address         | **35.178.87.119** |
| SSH Port           | 2200          |
| Username           | grader        |
| URL of Application | <a href="http://ec2-35-178-87-119.eu-west-2.compute.amazonaws.com/" target="_blank">http://ec2-35-178-87-119.eu-west-2.compute.amazonaws.com/</a> |

To connect to [Amazon Lightsail][4] Ubuntu instance you need the private ssh key `LightsailGraderPrivateKey-eu-west-2.pem` (supplied separately during the project submission process):
```
ssh -i ~/.ssh/LightsailGraderPrivateKey-eu-west-2.pem grader@35.178.87.119 -p 2200
```

## Summary of softwares installed

The following list of commands were used to install all the necessary software.
```shell
$ sudo apt-get update
$ sudo apt-get install -y software-properties-common python-software-properties
$ sudo add-apt-repository ppa:jonathonf/python-3.6
$ sudo apt-get update
$ sudo apt-get install -y python3.6 python-pip3 \
    apache2 libapache2-mod-wsgi-py3 \
    git ntp ufw fail2ban sendmail \
    postgresql postgresql-contrib
$ sudo apt-get upgrade
```

## Summary of configurations made

### 1. Update all currently installed packages
```shell
sudo apt-get update
sudo apt-get full-upgrade # Ubuntu 16.04
```

### 2. Change SSH port from 22 to 2200

1. Open the ssh config file:
`$ sudo vim /etc/ssh/sshd_config`
2. Change `Port 22` to `Port 2200`
3. Change `PermitRootLogin` from `without-password` to `no`.
4. Restart SSH service
`$ sudo service sshd restart`

### 3. Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

#### UFW
```shell
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow ntp
$ sudo ufw enable
```

Since we are using [Amazon Lightsail][4] for hosting our server instance, you would also have to go to [_Instances - Amazon Lightsail_][5] -> _Instances_ -> _Click on Your Instance_ -> _Networking_ -> _Firewall_ -> _Edit rules_, to manage your firewall settings so that you can control which ports are open to the public.

Here is an example on how your Firewall rules would look like:
![Amazon Lightsail Ubuntu Instance Firewall Settings](https://user-images.githubusercontent.com/13311878/37462341-9a347280-2877-11e8-8ba1-ee2f5992f370.png)

#### Fail2Ban
```shell
$ sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
$ sudo vim /etc/fail2ban/jail.local
```

* Set the following parameters in `/etc/fail2ban/jail.local`:

    + `backend = polling`
    + `destemail = YOURNAME@DOMAIN`
    + Under `[ssh]`

        ```shell
        enabled  = true
        port     = 2200
        filter   = sshd
        logpath  = /var/log/auth.log
        maxretry = 6
        ```

### 4. Give `grader` access

1. Create a new user account named `grader`.
```shell
$ sudo adduser grader
```

2. Give `grader` the permission to `sudo`.
```shell
$ sudo usermod -aG sudo grader
```

3. Create an SSH key pair for `grader` using the `ssh-keygen` tool.
```shell
$ su - grader
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
$ touch ~/.ssh/authorized_keys
$ ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ cat ~/.ssh/id_rsa # Copy contents of id_rsa to your local computer and save it.
$ rm ~/.ssh/id_rsa*
```

### 5. Configure the local timezone to UTC
```shell
$ sudo dpkg-reconfigure tzdata
```

### 6. Clone & Initialize the web application
```shell
$ sudo git clone https://github.com/rishi-ramawat/FSND_P4-Item_Catalog_Application.git /var/www/catalog-app
$ cd /var/www/catalog-app
$ bin/init_project # This script will install/upgrade all the required python packages for running this app.
```

### 7. Configuring PostgreSQL

1. Check that no remote connections are allowed (default):
    + Must have no remote wildcard like `0.0.0.0`, or specific IP like `122.123.124.125`.
    ```shell
    $ sudo vim /etc/postgresql/9.5/main/pg_hba.conf
    ```

2. Create a new database user named `catalog` that has limited permissions to the catalog application database.
```shell
#  switch to postgresql user
$ sudo su - postgres
$ createuser --interactive
   # Enter name of role to add: catalog
   # Shall the new role be a superuser? (y/n) n
   # Shall the new role be allowed to create databases? (y/n) n
   # Shall the new role be allowed to create more new roles? (y/n) n
   # CREATE ROLE
$ createdb item_catalogue
   # CREATE DATABASE
$ psql
postgres=# alter user catalog with encrypted password 'catalogPa$$w0rd123';
   # ALTER ROLE
postgres=# grant all privileges on database item_catalogue to catalog;
   # GRANT
postgres=# \q
# Switch back to grader
$ exit
```

### 8. Setting Up The `.env` File of Catalog App

Run `sudo vim /var/www/catalog-app/.env` and make sure:

+ The `DB_CONNECTION` variable is set to `pgsql` in the `.env` file.
+ The `DB_USERNAME` variable is set to `catalog`.
+ The `DB_PASSWORD` variable is set to `catalogPa$$w0rd123`.
+ The `DB_DATABASE` variable is set to `item_catalogue`.
+ The `GOOGLE_CLIENT_ID` & `GOOGLE_CLIENT_SECRET` are initialized by getting the credentials from https://console.developers.google.com/apis/credentials
+ The `FB_APP_ID` & `FB_APP_SECRET` are initialized by getting the credentials from https://developers.facebook.com/apps

### 9. Setting Up The Database

```shell
$ cd /var/www/catalog-app
$ sudo python3 app/models.py # Create all the tables in the database.
$ sudo python3 app/seeds.py  # Add some seed data into the database tables.
```

### 10. Create the `.wsgi` app

1. Enable the `wsgi` module
    ```shell
    $ sudo a2enmod wsgi
    ```

1. Create the `.wsgi` file
    ```shell
    $ sudo vim /var/www/catalog-app/catalog.wsgi
    ```

1. Copy and paste the following into the `catalog.wsgi`

    ```python
    #!/usr/bin/python3
    import sys
    import logging

    logging.basicConfig(stream=sys.stderr)
    sys.path.insert(0, "/var/www/catalog-app/app/")

    from project import app as application
    import config

    application.secret_key = config.APP_SECRET_KEY

    ```

### 11. Configure and enable a new Virtual Host

```shell
# Create the new virtualhost
$ sudo touch /etc/apache2/sites-available/catalog-app.conf
$ cat <<EOF | sudo tee -a /etc/apache2/sites-available/catalog-app.conf
<VirtualHost *:80>
        ServerName http://ec2-35-178-87-119.eu-west-2.compute.amazonaws.com/
        # ServerAdmin admin@mywebsite.com
        WSGIScriptAlias / /var/www/catalog-app/catalog.wsgi
        <Directory /var/www/catalog-app/app/>
            Order allow,deny
            Allow from all
        </Directory>
        Alias /static /var/www/catalog-app/app/static
        <Directory /var/www/catalog-app/app/static/>
            Order allow,deny
            Allow from all
        </Directory>
        ErrorLog ${APACHE_LOG_DIR}/error.log
        LogLevel warn
        CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
EOF
# Disable the default site, and enable the catalog app
$ sudo a2dissite 000-default
$ sudo a2ensite catalog-app
# Restart apache2 service
$ sudo service apache2 restart
```

### 12. Setup `.htaccess` file

```shell
$ sudo touch /var/www/catalog-app/app/.htaccess
$ cat <<EOF | sudo tee -a /var/www/catalog-app/app/.htaccess
Options -Indexes

RedirectMatch 404 /\.git

EOF
$ sudo service apache2 restart # Not generally required
```

[1]: https://www.udacity.com/course/full-stack-web-developer-nanodegree--nd004 "Udacity Nanodegree: Full Stack Web Developer"
[2]: https://www.udacity.com/course/linux-command-line-basics--ud595 "Linux Command Line Basics - Udacity"
[3]: https://www.udacity.com/course/configuring-linux-web-servers--ud299 "Configuring Linux Web Servers - Udacity"
[4]: https://lightsail.aws.amazon.com/ "Amazon Lightsail"
[5]: https://lightsail.aws.amazon.com/ls/webapp/home/instances "Instances - Amazon Lightsail"
