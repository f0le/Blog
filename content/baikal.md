+++
title = "CalDav/CardDav server with Baïkal"
date = 2021-05-28
+++

# Self hosted CalDav/CardDav server with Baïkal

First post after a long break. It is about self hosting a CalDav/CardDav server with Baïkal on OpenBSD.

From the [official website:](https://sabre.io/baikal/)

> Baïkal is a lightweight CalDAV+CardDAV server. It offers an extensive web interface with easy management of users, address books and calendars.
>
> Baïkal allows to seamlessly access your contacts and calendars from every device. It is compatible with iOS, Mac OS X, DAVx5 on Android, Mozilla Thunderbird and every other CalDAV and CardDAV capable application. Protect your privacy by hosting calendars and contacts yourself - with Baïkal.

<br></br>
## Requirements/Installation

Baikal requires a web server,PHP and a database. Install the needed packages with:

```sh
$ doas pkg_add baikal mariadb-server php-pdo_mysql php
```
By default OpenBSD installs Baïkal to `/var/www/baikal`.

## Configuration

For the database MariaDB is being used. Alternatively SQLite can also be used. For PHP choose which version suits your needs best. As for the web server, the OpenBSD built-in webserver is being used.

### Mariadb

If this is the first mariadb-server install, do the db initialization.

```sh
$ doas mysql_install_db
```

Secure the install.

```sh
$ doas mysql_secure_installation
```

Create a database, user and grant privileges on the db to the user.

```sql
create database baikal;
create user 'baikal'@'localhost' identified by '$password';
grant all privileges on baikal.* to 'baikal'@'localhost';
flush privileges;
```

Change the mysql socket in /etc/my.cnf, so the chrooted web server can access the db.

```cfg
[client-server]
socket = /var/www/var/run/mysql/mysql.sock
```

### PHP

Activate the database extension in the php.ini file.
```php7
extension=pdo_mysql
```

### Web server

The baïkal package on OpenBSD ships with a sample configuration for httpd. Adapt it for your needs.

```cfg
server "default" {
	listen on * port 80

	location "/.well-known/ca*dav" {
		block return 301 "http:///baikal/dav.php"
	}

	location "/baikal/*.php*" {
		root "/baikal/html"
		request strip 1
		fastcgi socket "/run/php-fpm.sock"
		directory index index.php
	}

	location "/baikal/*" {
		root "/baikal/html"
		request strip 1
		directory index index.php
	}
}
```

### Start Daemons

Enable/activate the daemons; 
MariaDB:
```sh
$ doas rcctl enable mysqld
$ doas rcctl start mysqld
```

Httpd:
```sh
$ doas rcctl enable httpd
$ doas rcctl start httpd
```

PHP:
```sh
$ doas rcctl enable php74_fpm
$ doas rcctl start php74_fpm
```

### Finish the install

Open your favorite webbrowser and browse to `http://$domain/baikal/admin/`

There, set a password for the admin account. Afterwards finish the install, with entering the db credentials.

&nbsp;

{{ resize_image(path="images/2021-05-26-134746_537x669_escrotum.png", width=800, height=800, op="fit") }}

{{ resize_image(path="images/db-pic.png", width=800, height=800, op="fit") }}

&nbsp;

### Calendars/address book

Create a new user. Use your email address as username for convenience. Finally create your calendars or address books under the user management tab.

&nbsp;

{{ resize_image(path="images/2021-05-26-134939_507x449_escrotum.png", width=800, height=800, op="fit") }}

{{ resize_image(path="images/2021-05-26-134718_1219x223_escrotum.png", width=800, height=800, op="fit") }}

&nbsp;

## Connecting with DAVx⁵

Now connect with the client of your choice. For android you can connect with [DAVx⁵.](https://www.davx5.com/tested-with/baikal)
