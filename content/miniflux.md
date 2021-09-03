+++
title = "Miniflux a minimalist feed reader on OpenBSD"
date = 2021-08-16
+++

# Self hosting Miniflux, a Rss feed reader

This blog post is about self hosting a minimalist feed reader named [Miniflux](https://miniflux.app/) . I prefer it over other feed readers because I like it's design philosophy. One thing I like especially is the sleek and fast UI that can be comfortably used on my mobile, even without an App. In fact it so pleasant to use on mobile that I prefer it over the Desktop UI, mainly because of it's support for touch events. Of course it is a free and open source software project as well. Some more things to note are, it disables pixel trackers by default, uses vanilla JS for the WebUI, is statically compiled and if your feed only displays a short snippet you can easily download the full original articles.

## Install necessary packages

Miniflux requires PostgreSQL for the db backend. The contrib package is needed for the required hstore extension.

```sh
$ doas pkg_add miniflux postgresql-server postgresql-contrib
```

After install have a look at the Readmes. I will be pasting some instructions from there.

```sh
$ cat /usr/local/share/doc/pkg-readmes/miniflux
$ cat /usr/local/share/doc/pkg-readmes/postgresql-server
```

## Configure Miniflux/Install PostgreSQL

First enable/start the PostgreSQL daemon.

```sh
$ doas rcctl enable postgresql
$ doas rcctl enable postgresql
```

For initializing the PostgreSQL DB I will quote the readme.

>If you are installing PostgreSQL for the first time, you have to create a default database first.  In the following example we install a database in /var/postgresql/data with a dba account 'postgres' and scram-sha-256 authentication. We will be prompted for a password to protect the dba account:

```sh
$ doas su - _postgresql
$ mkdir /var/postgresql/data
$ initdb -D /var/postgresql/data -U postgres -A scram-sha-256 -E UTF8 -W
```
Configure PostgreSQL. Create a user, a database and activate the hstore extension:

```sh
$ doas su - _postgresql
$ createuser -U postgres -P miniflux
$ createdb -U postgres -O miniflux miniflux
$ psql -U postgres miniflux -c 'create extension hstore'
> CREATE EXTENSION hstore;
```

Next change to the \_miniflux user, source the config file into the shell, run the database migration and create an admin user.

```sh
$ doas su -s/bin/sh - _miniflux
$ . /etc/miniflux.conf
$ miniflux -migrate
$ miniflux -create-admin
```

Now edit the config file, to make miniflux available over the web.

```cfg
LISTEN_ADDR=0.0.0.0:8080
BASE_URL=https://$domain/rss/
DATABASE_URL=user=dbuser password=dbpassword dbname=dbname sslmode=disable
CERT_FILE=/etc/letsencrypt/live/$domain/fullchain.pem
KEY_FILE=/etc/letsencrypt/live/$domain/privkey.pem
```

An alternative format for the db string is: `postgres://dbuser:dbpassword@localhost/dbname?sslmode=disable`. If your password contains a `^` the first schema will not work, so use the other one. More information on the db connection parameters is available [here.](https://pkg.go.dev/github.com/lib/pq?utm_source=godoc#hdr-Connection_String_Parameters)

Keep in mind, with every new Miniflux version you have to do a `miniflux -migrate`. If you want to do it automatically with every update, set `run migrations = 1` in `/etc/miniflux.conf`.

Enable/Start the daemon.

```sh
$ doas rcctl enable miniflux
$ doas rcctl start miniflux
```

### Using Let's encrypt/ Changing daemon variables

If you want to use Miniflux with Let encrypt certificates, the user to run the daemon under has to be changed. By default the user with which the Miniflux daemon is started is \_miniflux. For the daemon to be able access the certificate files root permissions are needed. In OpenBSD daemon variables are returned via `$ doas rcctl get $service`. So to get the Miniflux variables, type `$ doas rcctl get $service`. Simply enough these variables can be set with the `rcctl` utility as well, like so `$ doas rcctl set miniflux user root`. These changes would be overwritten by reboot. To make these changes permanent add `miniflux_user=root` to `/etc/rc.conf.local`. For more information on `rc.conf` and `rc.conf.local` the files where changes to daemon variables are stored and read from at boot time, read the excellent [manpage](https://man.openbsd.org/rc.conf.8).

<br></br>

## Create user/Add feeds

If you don't feel like using the admin account for everyday reading you can add another user via the settings. After that all that is left is to add some feeds and enjoy your minimalist feed aggregator.

For more information check out the [Miniflux docs](https://miniflux.app/docs/installation.html).
