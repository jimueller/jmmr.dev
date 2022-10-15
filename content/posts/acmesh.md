---
title: "acme.sh with lighttpd"
description: Switching from Certbot to acme.sh for certificates with lighttp on FreeBSD and DigitalOcean
date: 2022-10-15T13:40:43-05:00
categories:
  - hosting
tags:
  - freebsd
  - letsencrypt
  - lighttpd
slug: acmesh
---

After a FreeBSD upgrade seemed to break my Certbot certificate renewal process, I decided to switch to use [acme.sh](https://acme.sh/) instead. The process was pretty straightfoward and I like the idea of just using a basic shell script to manage certificates.

# Install acme.sh

This step was simple, using the `curl` method.

```sh
curl https://get.acme.sh | sh -s email=my@example.com
```

# Issue a cert

This site is a FreeBSD droplet on Digital Ocean using Digital Ocean DNS.

Followed the [acme.sh Digitial Ocean DNS directions](https://github.com/acmesh-official/acme.sh/wiki/dnsapi#20-use-digitalocean-api-native) to generate and set an API key.

Then issued a cert.

```sh
acme.sh --issue --dns dns_dgon -d jmmr.dev -d www.jmmr.dev
```

# Install the Cert for lighttpd

Once the certs are issued, these need to be copied to where lighttpd is looking for them. First thing is to confirm where this is by checking the `lighttpd.conf` file's `ssl.privkey` and `ssl.pemfile` settings.

```
##
##  SSL Support
## -------------
##

ssl.engine = "enable"

# Let's Encrypt certs (acme.sh)
ssl.privkey = "/path/to/jmmr.dev/privkey.pem"
ssl.pemfile = "/path/to/jmmr.dev/fullchain.pem"
```

Now we can run the acme.sh `--install-cert command to perform the copy. It's also important to specify the `--reloadcmd` to ensure lighttpd picks up the new certs after they're copied. I'm just restarting lighttpd since it's fast, it's just hosting static files, and there's not enough traffic to care about dropping connections. It may be possible to more gracefully reload the certs if needed.

```sh
acme.sh --install-cert -d jmmr.dev \
--key-file       /path/to/jmmr.dev/privkey.pem  \
--fullchain-file /path/to/jmmr.dev/fullchain.pem \
--reloadcmd     "service lighttpd restart"
```

# That's it

acme.sh installed a cron job to check daily and renew certs if needed.
