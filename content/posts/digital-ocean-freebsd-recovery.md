---
title: "Digital Ocean Freebsd Recovery"
date: 2022-12-30T13:45:49-06:00
categories:
 - admin
tags:
 - digitalocean
 - freebsd
slug: digital-ocean-freebsd-recovery
---

I was recently configuring FreeBSD to send mail destined for `root` to my external email 
using [SSMTP](https://www.freebsd.org/cgi/man.cgi?query=ssmtp&sektion=8&manpath=freebsd-release-ports).
That worked fine, but in the process I messed up the permissions to my home directory. I didn't 
realize it until I tried to ssh into my server and it was falling back to password authentication!

To my surprise, it turns out that `PasswordAuthentication no` in the `sshd_config` doesn't actually disable
password authentication if PAM allows it.  I needed to set `ChallengeResponseAuthentication no` as well.
Digital Ocean has a [nice guide](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-freebsd-server) so I won't repeat that here.
Anyway, this was all a blessing in disguise to realize that my public facing server would still
accept password ssh logins.

Back to the point of this post, I had setup the FreeBSD Droplet back when Digital Ocean offered
that. It had automatically uploaded my ssh key to the server so I never knew what the account password
was.  Digital Ocean in the mean time stopped offering password resets through their web UI for FreeBSD.
So I needed a way to get back into my server.  I knew I could reset passwords if I could get to the
single user mode.

Luckily, [Admin...by accident](https://www.adminbyaccident.com/) had a nice write up on continuing
to use FreeBSD on Digital Ocean that had a section on [how to get to single user mode](https://www.adminbyaccident.com/freebsd/how-to-upload-a-freebsd-custom-image-on-digitalocean/).

For quick reference, the steps are

1. Open the **Console** for the droplet, it should show a login prompt and maybe some logs.
1. **Reboot** the droplet from **Power** > **Power cycle**
1. Go back to the **Console** window and watch the startup.
1. The FreeBSD menu will display, and choose **Option 2** to **Boot Single user**
1. Mount the file system as read-write with `mount –o rw –u /`

At this point you can reset a user password with the `passwd` utility.
