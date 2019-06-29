# Project: Linux Server Configuration

This is the sixth project of the Udacity's Full Stack Web Developers Nanodegree.

The project consists in take a baseline installation of a Linux server and prepare it to host a web application. It is part of this project secure the server from a number of attack vectors, install and configure a database server, and deploy the [Item Catalog Project](https://github.com/rccmodena/catalog). This project slightly modified the original project, changing the source code for Python 3 and changing the database from SQLite to PostgreSQL.

## Amazon Lightsail

The Linux server instance choosed was Amazon Lightsail, with the following specifications:

- Name: Ubuntu-Linux-Configuration-Server
- Linux: Ubuntu 18.04 LTS
- IP: 3.220.69.7
- SSH Port: 2200
- Reviewer user: grader

## Server installation and configuration

- Create AWS account [https://aws.amazon.com/](https://aws.amazon.com/);
- Create IAM Admin User [(reference)](https://docs.aws.amazon.com/IAM/latest/UserGuide/getting-started_create-admin-group.html);
- Create IAM User for Lightsail, with the correct permissions [(reference)](https://lightsail.aws.amazon.com/ls/docs/en_us/articles/amazon-lightsail-managing-access-for-an-iam-user);
- Create a new Lightsail Instance [https://lightsail.aws.amazon.com/ls/webapp/create/instance](https://lightsail.aws.amazon.com/ls/webapp/create/instance)
  - Select a plataform: Linux/Unix;
  - Select a blueprint: OS Only - Ubuntu 18.04 LTS;
- In the network tab:
  - Create Static IP;
  - Firewall, allow the following connections:
    - SSH - TCP - 22;
    - HTTP - TCP - 80;
    - Custom - TCP - 123;

- Connect to the server using SSH:

```sh
$ ssh -i ~/.ssh/<NAME_OF_THE_KEY_DOWNLOADED_FROM_AWS>.pem ubuntu@3.220.69.7
```

- Update available package list:

```sh
$ sudo apt-get update
```

- Upgrade installed packages:

```sh
$ sudo apt-get upgrade
```

- Create the Udacity reviewer account:

```sh
$ sudo adduser grader
```

- Give grader the permission to sudo:

```sh
$ printf "# CLOUD_IMG: This file was created/modified by the Cloud Image building process\ngrader ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/grader
```

- Create an SSH key pair for grader using the ssh-keygen tool. In my machine I run:

```sh
$ ssh-keygen
```

```sh
$ su - grader
$ mkdir .ssh
$ printf "<PUT HERE THE CONTENT OF THE PUBLIC KEY>" | sudo tee .ssh/authorized_keys
$ sudo chmod 700 .ssh
$ sudo chmod 644 .ssh/authorized_keys
```

Make the following changes in the file /etc/ssh/sshd_config:
- Change the SSH port from 22 to 2200.
- Forcing Key Based Authentication;
- Disable remote login of the root user;

```sh
Port 2200
PasswordAuthentication no
PermitRootLogin no
```

- Restart the service:

```sh
$ sudo service ssh restart
```

- Configure the Uncomplicated Firewall (UFW) to only allow incoming connections for SSH (port 2200), HTTP (port 80), and NTP (port 123).

```sh
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow 2200/tcp
$ sudo ufw allow www
$ sudo ufw allow 123/tcp
$ sudo ufw enable
```

- Add the port 2200, and remove the port 22 in the Lighsail console Firewall:
    - HTTP - TCP - 80;
    - Custom - TCP - 123;
    - Custom - TCP - 2200;

- Configure the local timezone to UTC.

```sh
$ sudo dpkg-reconfigure tzdata

Select: None of the above
Select: UTC
```

## Apache installation and configuration

- Install Apache and mod_wsgi module.

```sh
$ sudo apt-get install apache2
$ sudo apt-get install apache2-dev
$ sudo apt-get install libapache2-mod-wsgi-py3
```

- Configure apache file /etc/apache2/sites-enabled/000-default.conf ([reference](https://modwsgi.readthedocs.io/en/develop/user-guides/configuration-guidelines.html)):

```sh
<VirtualHost *:80>
  ServerName 3.220.69.7
  ServerAdmin rudi.modena@gmail.com
  DocumentRoot /var/www/html

  ErrorLog ${APACHE_LOG_DIR}/error.log
  CustomLog ${APACHE_LOG_DIR}/access.log combined

  WSGIScriptAlias / /var/www/catalog/app.wsgi

  <Directory /var/www/catalog>
  Require all granted
  </Directory>
</VirtualHost>

```

- Restart Apache:

```sh
$ sudo apache2ctl restart
```

## Configuration of the Item Catalog project

- Install pip3.

```sh
$ sudo apt install python3-pip
```

- Install virtualenv.

```sh
$ sudo pip3 install virtualenv
```

- Install PostgreSQL:

```sh
$ sudo apt-get install postgresql postgresql-contrib libpq-dev
```

- Clone the repository with the Item Calog Project:

```sh
$ cd /var/www/
$ sudo git clone https://github.com/rccmodena/catalog.git
$ sudo chown -R grader catalog/
```

- Configure virtualenv and install all Python packages needed for the app:

```sh
$ cd catalog
$ sudo virtualenv venv
$ sudo chown -R grader venv/
$ source venv/bin/activate
$ pip3 install wheel
$ pip3 install Flask
$ pip3 install SQLAlchemy
$ pip3 install Flask-SQLAlchemy
$ pip3 install oauth2client
$ pip3 install requests
$ pip3 install psycopg2
```

- Configure PostgreSQL and create database catalog:

```sh
$ sudo -i -u postgres
$ createuser catalog_user
$ createdb catalog
$ psql
postgres=# alter user catalog_user with encrypted password 'catalog2019';
```

- For Google OAuth2 it necessary to give the host name, so it was used the magic domain ([http://xip.io/](http://xip.io/)), http://3.220.69.7.xip.io. ([reference]((https://stackoverflow.com/questions/49779283/permission-denied-to-generate-login-hint-for-target-domain-when-hosted-on-aws))

- Create a file inside the folder /var/www/catalog/catalog with the name cliente_secrets.json. Put the following content inside:

```sh
{"web":{"client_id":"PUT_HERE_THE_GOOGLE_CLIENT_ID","project_id":"PUT_HERE_THE_PROJECT_ID","auth_uri":"https://accounts.google.com/o/oauth2/auth","token_uri":"https://oauth2.googleapis.com/token","auth_provider_x509_cert_url":"https://www.googleapis.com/oauth2/v2/certs","client_secret":"PUT_HERE_THE CLIENT_SECRET","redirect_uris":["http://3.220.69.7.xip.io"],"javascript_origins":["http://3.220.69.7.xip.io"]}}
```

- Add the host http://3.220.69.7.xip.io to the authorized JavaScript origins in the Google Developers Console.

- Restart Apache:

```sh
$ sudo apache2ctl restart
```

**Now Item Catalog App is up and running!!!**

## License

The contents of this repository are covered under the [MIT License](LICENSE).
