---
title: "FreeBSD Admin"
description: "Notes on administering FreeBSD"
date: 2021-09-25T16:39:00-06:00
categories:
  - freebsd
slug: freebsd-admin
---

# Upgrades

FreeBSD uses `freebsd-update` utility to perform security updates and upgrading to releases. Instructions are on the [FreeBSD Handbook](https://docs.freebsd.org/en/books/handbook/cutting-edge/#updating-upgrading-freebsdupdate).

## Security Updates

```
freebsd-update fetch
freebsd-update install
```

## Minor Version Updates

Check for current [releases](https://www.freebsd.org/releases/)

Example for 12.2

```
; Update current version
freebsd-update fetch
freebsd-update install

; Upgrade kernel
freebsd-update upgrade -r 12.2-RELEASE
freebsd-update install

; Reboot
shutdown -r now

; Upgrade userland
freebsd-update install

; Remove old system libraries
freebsd-update install

; Reboot
shutdown-update install
```

# Managing Services (init and rc)

lighttpd example

```sh
# enable startup
sudo sysrc lighttpd_enable=YES

# start
sudo service lighttpd start

# stop
sudo service lighttpd stop

# restart
sudo service lighttpd restart
```
