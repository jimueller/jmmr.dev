---
title: "OpenBSD Admin"
description: "Notes on administering OpenBSD"
date: 2024-04-20T16:20:00-05:00
categories:
- openbsd
slug: openbsd-admin
---

# Upgrades

OpenBSD uses [sysupgrade](https://man.openbsd.org/sysupgrade) for unattended upgrades, this is the easiest way.

After the upgrade, remember to update packages with `pkg_add -u`

## Patches

Run `syspatch`
