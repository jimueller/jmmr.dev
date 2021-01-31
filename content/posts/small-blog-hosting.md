---
title: "Small Blog Hosting"
description: "Setting up a small, performant static Hugo generated website on a Digitial Ocean FreeBSD droplet, with LetsEncrypt SSL, Lighttpd, and Compression"
date: 2020-08-30T19:39:29-05:00
categories:
  - hosting
tags:
  - freebsd
  - digitalocean
  - letsencrypt
  - lighttpd
slug: small-blog-hosting
draft: false
---

Always inspired by a post on [tumbledry.org about efficient performance from a small machine](https://tumbledry.org/2011/08/31/9_million_hits_day_with_120) I decided to begin this site mostly as a way to play around with optimizing for performance and security. Additionally, while I use Linux (Fedora and Ubuntu) everyday I've always been interested in using true unix systems. I've used FreeBSD on VMs a few times, but I decided to try hosting a blog with it. Mainly to learn about the differences and similarites between a BSD and linux.

The setup so far.

- small FreeBSD droplet on DigitalOcean (raspberry pi someday?)
- [lighttpd](https://www.lighttpd.net/) server, mainly because it has light in the name and I've never used it before
- [letsencrypt](https://letsencrypt.org/) for tls via [certbot](https://certbot.eff.org/lets-encrypt/freebsd-other)
- hugo for the blog

## Digital Ocean

Created a FreeBSD 12.1 zfs small shared droplet with 1GB of ram and 25 DB SSD.

## FreeBSD

Setup SSH and a ipfw following digital ocean's guides, which I find really handy. Switch the shell to zsh and installed a few programs from pkg and building from source via ports. Building from source was interesting, it did take longer on the small droplet, but it just worked. I'm sure there are pros and cons, but it was pretty straight forward.

## lighttpd

I've heard of lighttpd but never used it before. I've used Apache and nginx before and they're fine, but I thought this would be a nice chance to try another server. After reading more, it was cool to see it started as an experiment known as the [C10K problem](http://www.kegel.com/c10k.html) in the late '90s.

### TLS

After installing, I configured lighttpd to listen only on 443 since I don't plan on serving http. I went about installing certbot and pointing lighttpd to the certs. There seems to be fairly outdated info on this that are a little more complex. I'm using lighttpd 1.44 and was able to just point to the fullchain.pem and privkey.pem from lighttpd.conf, no combining certs required. This makes it a bit easier to configure the renewal via cron. lighttpd starts as root, then switches to an unprivilidged account. When it is starting it uses it's root privilidges to bind to the ports, but it also means that it can access the letsencrypt certs without messing with their permissions.

I decided to checkout the results on [ssllabs.com](https://ssllabs.com) and got a grade B just using the lighttpd defaults. Not bad, but I decided to try to get the best I could. I didn't capture the results, but the main issues were no CAA, TLS 1.0 and TLS 1.1 support, DH something, and supporting weak ciphers.

Fixing the CAA was easy, just added the appropriate DNS CAA records so I state that no other CA should be generating certs for my domain. It was also straight forward to remove support for TLSv1.0 and TLSv1.1 through the lighttpd configs. That left me with a DH param issue and weak ciphers. It would have been simple to just only support TLSv1.3 which would have crossed-off those issues, but I wanted a bit of a challange to support TLSv1.2 securely. I didn't feel like going through the giant list of ciphers, so I used [mozilla's awesome SSL Configuration Generator](https://ssl-config.mozilla.org/#server=lighttpd&version=1.4.55&config=intermediate&openssl=1.1.1d&guideline=5.6). This generated an appropriate cipher-list.

This finally resulted in an [A+ grade](https://www.ssllabs.com/ssltest/analyze.html?d=jmmr.dev).

### compression

I configured mod_deflate on lighttpd using the default settings. I did not setup a cache for compressed files, which will be on the todo list.
