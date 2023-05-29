---
title: "Openbsd Static Hosting"
date: 2023-05-06T13:53:37-05:00
categories:
 - category
tags:
 - lighttpd
 - openbsd
slug: openbsd-static-hosting 
---

## Installing Lighttpd

Lighttpd is available as a package

```
pkg_add lighttpd
```

Choose the version with just lighttpd, likely option 1

## Configure

Config file is at `/etc/lighttpd.conf`

Lighttpd will be chrooted to `/var/www` so config paths are relative to that.

Change `server.document-root` to `"/htdocs/"

Check that the service will start

```
rcctl -df start lighttpd
```

You may see the following error on OpenBSD 6.4 or greater.

```
daemonized server failed to start; check error log for details
```

And the `/var/www/logs/error.log` shows:

```
2023-05-06 18:37:43: (configfile.c.1771) opening /dev/null failed: No such file or directory
2023-05-06 18:37:43: (server.c.1584) Opening errorlog failed. Going down.
```

The reason is the chroot to `/var/www`, which means `/dev/null` is no longer visible to the process.  You have
to make a fake `/dev/null`.

Luckily, I found [a post on daemonforums](https://daemonforums.org/showthread.php?t=11791) which linked to the the solution on [Reddit](https://www.reddit.com/r/openbsd/comments/nygjdm/lighttpd_cant_find_devnull_on_69/h1kl7ge/).

The steps I followed were 1-4 from that post.  There was no `nodev` restriction for `/var`

    1. change into that directory

    ```
    # cd /var/lighttpd/chroot/
    ```

    2. create a dev/ directory in there

    ```
    # mkdir dev
    ```

    3. set the permissions on it
    
    ```
    # chmod 755 dev
    # chown root:wheel dev
    ```

    4. make the expected null device in there and make it world-accessible

    ```
    mknod dev/null c 2 2
    chmod 666 dev/null
    ```

Re-running the rcctl start command was successful.

Next, enable lighttpd to start at boot

```
rcctl enable lighttpd
```


