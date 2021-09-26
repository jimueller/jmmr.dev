---
title: "Installing Miniflux on FreeBSD"
description: "Installing Miniflux RSS reader on FreeBSD with Postgres, Lighttpd, and LetsEncrypt"
date: 2021-09-26T13::35-05:00
categories:
  - admin
tags:
  - miniflux
  - lighttpd
  - postgres
  - freebsd
slug: miniflux-on-freebsd
---

I don't use RSS readers much these days, not since Google killed Reader. However, I missed having a way to subscribe to blogs and be able to keep tabs on what's going on. Somewhere I came across [miniflux](https://miniflux.app) and I liked it's style and minimal and responsive design.

I decided it would be good to host this on my Digital Ocean droplet that's hosting this site, since I don't really want to host anything public from my home network. Also, the blog doesn't get much (any? its ok) traffic so that should be no issue.

# Create Sub-domian

First step was to setup a subdomain to access it from. I went to the Digital Ocean control panel and created a new A record for the subdomain (rdr.jmmr.dev) pointing to my droplet. Often this is easier to manage when setting up a reverse proxy, but it does look like miniflux can handle it. Even so, I hadn't used lighttpd as a reverse proxy before, so I'd prefer to keep it simple. In hindsight, this probably would have been fine.

# Install Postgres

Miniflux uses postgres, so I installed the postgres packages required. Miniflux does need the hstore extension which is in the contrib package and won't be in most "postgres install" references, so don't forget that one. I first searched `pkg` for the postgresql13 packages (latest stable version)

```sh
pkg search postgresql13

pgtcl-postgresql13-2.1.1_2     TCL extension for accessing a PostgreSQL server (PGTCL-NG)
postgresql13-client-13.3       PostgreSQL database (client)
postgresql13-contrib-13.3      The contrib utilities from the PostgreSQL distribution
postgresql13-docs-13.3         The PostgreSQL documentation set
postgresql13-plperl-13.3       Write SQL functions for PostgreSQL using Perl5
postgresql13-plpython-13.3     Module for using Python to write SQL functions
postgresql13-pltcl-13.3        Module for using Tcl to write SQL functions
postgresql13-server-13.3_1     PostgreSQL is the most advanced open-source database available anywhere
```

I installed the three required packages miniflux needs for postgres

```sh
sudo pkg install postgresql13-server-13.3_1 postgresql13-contrib-13.3 postgresql13.3
```

Next I initialized postgres...

```sh
sudo /usr/local/etc/rc.d/postgresql initdb
```

...and configured it to start automatically...

```sh
sudo sysrc postgresql_enable=YES
```

... and finally started it

```sh
sudo service postgresql start
```

# Install miniflux and configure DB

And then installed miniflux (this probably can be done at the same time as postgres)

```sh
sudo pkg install miniflux-2.0.32
```

**Error** for some reason, I caught an error that the "pw: user 'miniflux' disappeared during update", after some research, I found the pw db needed to be sync'd with the following command.

```sh
sudo pwd_mkdb -p /etc/master.passwd
```

After this, I re-ran the pkg install for miniflux and there was no error.

Following the [database configuration instructions for miniflux](https://miniflux.app/docs/installation.html#database).

```sh
su - postgres

createuser -P miniflux
Enter password for new role: ******
Enter it again: ******

createdb -O miniflux miniflux

psql miniflux -c 'create extension hstore'
CREATE EXTENSION
```

There are a couple of options to add the enable the HSTORE extension, I went with the route of temporarily granting SUPERUSER permissions to the miniflux user.

**Remember to change this back after running migrations**

```sql
ALTER USER miniflux WITH SUPERUSER;
```

Next I added the database and base url params in the miniflux config file at `/usr/local/etc/miniflux.env`. This is the file automatically passed to miniflux when starting via init.

```ini
DATABASE_URL=dbname=miniflux sslmode=disable user=miniflux password=THE_PASSWORD
BASE_URL=https://rdr.jmmr.dev
LISTEN_ADDR=0.0.0.0:8080
```

Then run the migrations to setup the database, pointing it to the config file so it can get the DATABASE_URL.

```sh
miniflux -c /usr/local/etc/miniflux.env -migrate
```

**Important - revoke SUPERUSER permissions from miniflux user, if you granted them**

Switch to the postgres user and disable SUPERUSER for miniflux user

```
su - postgres
psql ALTER USER miniflux WITH NOSUPERUSER;
```

Switch back to your user account

And create an admin user

```sh
miniflux -c /usr/local/etc/miniflux.env -create-admin
```

At this point, miniflux should be good to go, so I enabled it to start automatically and started it up.

```sh
sudo sysrc enable_miniflux=YES
sudo service miniflux start
```

# Configure lighttpd to reverse proxy requests to miniflux

Lighttpd is already running this blog on jmmr.dev, so I needed to proxy request for `rdr.jmmr.dev` to miniflux. Lighttpd has [mod_proxy](https://redmine.lighttpd.net/projects/1/wiki/Docs_ModProxy) for this task.

First I enabled mod_proxy in the config file (/usr/local/etc/lighttpd/lighttpd.conf)

```
server.modules += ( "mod_proxy" )
```

And then configured to proxy based on the host header to miniflux.

```
#######################################################################
##
## mod_proxy
## Proxy Configuration
##
##
$HTTP["host"] == "rdr.jmmr.dev" {
  proxy.server = ("" =>
    (
      (
        "host" => "127.0.0.1",
        "port" => "8080"
      )
    )
  )
}
```

And finally restarted lighttpd to pick up the new config.

```
sudo service lighttpd restart
```

# Let's Encrypt

Next I needed to update my LE cert to also be valid for the new subdomain. Let's encrypt has a handy command, expand, to add an additional DNS name to an existing configuration.

```sh
certbot --expand -d jmmr.dev,rdr.jmmr.dev
```

At this point, certbot had generated a new cert, and lighttpd needs a restart to pick it up.

```sh
sudo service lighttpd restart
```

# Final setup

Miniflux was now being served up via https. I logged in with the admin user and created a non-admin user for myself. I then imported an OPML feed I had backed up from Google Reader.
