---
layout: post
title:  "MySQL to Postgres migration"
date:   2018-11-01 21:42:29 -0700
categories: posts
---

# Migration from MySQL 5.7 to Postgres 9.5

### Background
This notebook documents the complete process of transfering a database from MySQL to Postgres. I am also choosing to manage the database using the lightweight but reasonably full featured UI [adminer](https://www.adminer.org/). This requires the additional step of installing a web server annd configuring it to run `admirer.php`.

### System configuration
The database and webserver run on the same server.
- Database:  From *MySQL 5.7* to *Postgres 9.5*
- Database OS:  Ubuntu 16.04

### Procedure
Install Postgres
```
sudo apt-get update && sudo apt-get upgrade
sudo apt-get install postgresql postgresql-contrib

## Password for postgres account
sudo passwd postgres
```

Create another user

**Warning**:  If you use an existing OS user, postgres will take over ownership of that user's home directory. To avoid this, create a separate user for the DB.
```
## Another user
sudo -u postgres createuser <username>
sudo -u postgres createdb <dbname>

$ sudo -u postgres psql
psql=# alter user <username> with encrypted password '<password>';

psql=# grant all privileges on database <dbname> to <username> ;
```

Move the database directory (if needed)
```
## check the current directory
sudo -u postgres psql

postgres=# show data_directory;
postgres=# \q

sudo systemctl stop postgresql
sudo systemctl status postgresql

sudo rsync -av /var/lib/postgresql /home/datascience
sudo mv /var/lib/postgresql/9.5/main /var/lib/postgresql/9.5/main.bak

## Edit the conf file
sudo vi /etc/postgresql/9.5/main/postgresql.conf
. . .
data_directory = '/home/datascience/postgresql/9.5/main'
. . .

## Restart Postgres
sudo systemctl start postgresql
sudo systemctl status postgresql

sudo -u postgres psql
postgres=# show data_directory;

```

Install apache2 and adminer - This is used to manage the database on a web browser
[Adminer HOWTO](https://www.techrepublic.com/article/how-to-make-mysql-administration-simple-with-adminer/)
```
sudo apt-get install apache2

## PHP libraries
sudo apt-get install php
sudo apt-get install php7.0-curl php7.0-gd  php7.0-mcrypt php7.0-pgsql php7.0-xml libapache2-mod-php7.0
# sudo apt-get install php-pear php7.0-dev php7.0-zip php7.0-curl php7.0-gd php7.0-mysql php7.0-mcrypt php7.0-xml libapache2-mod-php7.0

## adminer
sudo mkdir /usr/share/adminer
cd /usr/share/adminer
sudo wget "http://www.adminer.org/latest.php" -O /usr/share/adminer/latest.php
sudo ln -s /usr/share/adminer/latest.php /usr/share/adminer/adminer.php
sudo echo "Alias /adminer.php /usr/share/adminer/adminer.php" | sudo tee /etc/apache2/conf-available/adminer.conf
sudo a2enconf adminer.conf
sudo systemctl restart apache2
```

Install [pgloader](https://github.com/dimitri/pgloader)
```
## Build from source to handle an error with MySql 5.7
sudo apt-get install sbcl unzip libsqlite3-dev make curl gawk freetds-dev libzip-dev
mkdir pgloaderbuild
cd pgloaderbuild
wget https://github.com/dimitri/pgloader/archive/master.zip
unzip master.zip
cd pgloader-master/
make pgloader
./build/bin/pgloader --help
```

Transfer the MySql database to Postgres
In my case, I ran the pgloader program from the 'postgres' account
```
## Might have to create the database in the Postgres database
## Note:  Run this from the same Linux account that owns the database (in this case analytics)
./build/bin/pgloader mysql://root:passwd@localhost/analytics postgresql://postgres:passwd@localhost/analytics
```

Log in to adminer Web UI: `http://webserver/adminer.php`

## References
- [https://medium.com/@Riverside/how-to-install-apache-php-postgresql-lapp-on-ubuntu-16-04-adb00042c45d]
- [https://www.techrepublic.com/article/how-to-make-mysql-administration-simple-with-adminer]
- [https://www.digitalocean.com/community/tutorials/how-to-install-the-apache-web-server-on-ubuntu-16-04]
- [https://www.adminer.org]
- [https://github.com/dimitri/pgloader]
- [https://www.digitalocean.com/community/tutorials/how-to-move-a-postgresql-data-directory-to-a-new-location-on-ubuntu-16-04]
- [https://medium.com/coding-blocks/creating-user-database-and-adding-access-on-postgresql-8bfcd2f4a91e]
