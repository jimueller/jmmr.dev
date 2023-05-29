---
title: "Switching to Openbsd and Vultr"
date: 2023-05-29T12:26:18-05:00
categories:
 - hosting
tags:
 - openbsd
 - vultr
slug: switching-to-openbsd-and-vultr
---

Digital Ocean stopped supporting FreeBSD in 2022, and although my droplet would continue to work, it was more challenging to
to use since some of the admin functions were removed. I decided to switch this site over to Vultr since they support BSDs and
thought it would be a good opportunity to try out OpenBSD. I wasn't overly looking forward to switching everythign over, but it wasn't too painful.

## New VM

I had never heard of Vultr before, but they seem pretty legit.  I went with their AMD 25 GB NVMe, 1 vCPU, 1 GB Memory VM for $6/month.  It was tempting to go with a cheaper option since even this is a bit overkill for this site, but the savings was only $1/month.  Now it appears they have an even cheaper $3.50 Intel option with about half the specs.  It would be interesting to see what performance could be squeezed out of that with OpenBSD.

The first thing to do was setup the OpenBSD box, there wasn't much to do, it's known for sane security out of the box and it had a pretty good default pf.conf allowign ssh, http, and https. I did harden up the `sshd` settings to disable password logins and disable root.  

I tried installing lighttpd, but ran into some errors where it wouldn't start up.  I decided it would be a nice opportunity to learn OpenBSD's built-in webserver, httpd and proxy (relayd).  The OpenBSD philosophy of simple, hard to mess up configs was really on display here.  It was basically just putting in the DNS name and it was good to go, including Let's Encrypt, which is built-in. (It was almost no fun since the main point of this site is to learn some BSD and different web servers).

Next step was to cut DNS over.  I was using Digital Ocean's DNS with this domain.  I decided to switch over to Cloudflare, just so it's one less thing to change if I switch hosting providers again.  I am not using the proxy functionality because I want to test out the performance of the box itself and don't want to use any of Cloudflare's caching functionality.  Obviously if this were a high-traffic site things would be different.

Once the DNS was switched over, which only took a minute or two, the new site was serving traffic and I deleted my Digital Ocean account.  It was disappointing that they removed official BSD support, but I get it, probably not enough revenue to justify the support.  I did enjoy using Digital Ocean and found it easy to use, although Vultr seems on par.

The final piece was updating the CI/CD pipeline.  I had been using CircleCI with GitHub.  I found it pretty easy to use and a fine product, but this was a chance to try out GitHub Actions.  I am pretty familiar with GitLab CI so I had wanted to see how GHA compared.

I started with the build example from Hugo's site and removed all of the GitLab Pages steps.  The build worked fine.

The next step was getting the files to the server.  With my FreeBSD box, I had created a specific CI/CD user and used the ACLs to allow it to write to the site directory without adding it to the owning group.  This seemed more secure in case the credentials were ever leaked, but I'm not sure if that's a reality.  It was fun learing about the ACLs anyway.  OpenBSD doesn't do ACLs so I added the CI/CD user to the owning group.  I plan to look into further restricting the user, but the current setup is secure enough for now.

I initially thought that OpenBSD didn't have `rsync` since that is GPL'd, so I started using `scp`.  I ran into a few issues with the SCP GitHub Action in that it was geared toward the Linux implementation and used a few flags that OpenBSD's version didn't support.  I researched a bit more and discovered that OpenBSD does have `rsync` though `openrsync`.  I used an `rsync` GitHub Action to ship the files to the server.


## OpenBSD Thoughts

- Removing the `sudo` muscle memory for `doas` is hard, but at least I know what `doas` means...
- Almost too easy to use `httpd` and `relayd` to securely serve up a site
- `httpd` is has few configuration options, like per file or MIMEType caching options.  It also doesn't have on the fly compression, but I get that's the Unix philosophy and it would be easy enough to gzip the files as part of the CI/CD process or as a cron job.

## GitHub Actions Thoughts

- It seems to have an NPM mentality of "just pull in some action some random person provides" which I don't really care for putting that sort of trust into someone else's docker image, especially when dealing with SSH keys, etc.  I ended up doing that for rsync, but I suspect I can just run the rsync command directly from a "script" tag with a base Ubuntu image like I would with GitLab CI.  That's something to also look into.

## Todos

- Check if I can further restrict what the CI/CD user can do.  Perhaps use `doas` to only allow it to write to the site directory and rsync, i.e. don't allow it to otherwise login or run other commands.
- Gzip files in GitHub actions
- Remove pre-built rsync GitHub Action and replace with plain old bash script.
