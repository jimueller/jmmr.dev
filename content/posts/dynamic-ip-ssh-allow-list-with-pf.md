---
title: "Dynamic Ip SSH Allow List With PF"
date: 2024-05-04T17:52:02-05:00
categories:
  - admin
  - openbsd
  - pf
tags:
  - openbsd
  - pf
slug: dynamic-ip-ssh-allow-list-with-pf
draft: false
---

This site is hosted on a Vultr OpenBSD VM and I SSH into it to perform maintenance. As with anything running on the Internet, it's going to get a lot of traffic trying to get in, especially SSH. Even though I have implemented properly hardened SSH settings (no root, no passwords, etc) I really wanted to go a step further and only allow SSH from my home connection.

## Using sshd_config

With `/etc/ssh/sshd_config`, it is possible to use a `Match` directive to allow as user to only login from certain IP addresses.

```
Match User someuser
  AllowUsers someuser@10.10.10.10
```

This worked well with my old ISP as even though the IP was in theory dynamic, it never changed in over two years. Also, it was a small ISP, and they only had one IP block, so I just allow listed the entire block.

However, after moving, I had to switch to a national ISP with many address blocks and the IP changed all of the time. Whenever it did, I'd have to console into the server and update the IP address.

I thought about writing a script to check the IP address from dynamic DNS and update the `sshd_config` and restart sshd, but there was an easier way.

## Using PF

Since the server runs OpenBSD, I have the awesome `pf` packet filter firewall at my disposal. One feature of pf is tables which will take in a hostname and can be dynamically updated.

The first thing to do is to backup the pf configuration file in `/etc/pf.conf` in case I mess this up. Secondly, even though I'm performing the following steps over an SSH connection, I know I can use the Vultr console if I lock myself out.

The first step was to setup a table for allowed ssh ips. On boot, pf correctly starts before DNS services so it's not possible to put a domain directly into `pf.conf`. The solution is to use a persistent table.

```
# Allowed ssh ips
table <allowssh> persist
```

Then use this table in a rule to allow ssh

```
# Rule to allow ssh
pass in proto tcp from <allowssh> to port ssh
```

Validate the changes

```
# validate
pfctl -nf /etc/pf.conf
# load
pfctl -f /etc/pf.conf
```

Now I need to populate the table dynamically. To do this I created a script called `pf_allowssh_update.sh`

```
touch pf_allowssh_update.sh
chmod +x pf_allowssh_update.sh
```

Then add the following

```sh
#!/bin/sh
#
#########################################################
# Flushes and updates allowed ssh clients
#########################################################

# Flush the table of all addresses to get rid of the old one
pfctl -t allowssh -T flush
# Add allowed
pfctl -t allowssh -T add dyndns.example.com
```

As mentioned in the comments, the first step is to use pfctl to empty the list to clear out the old IP address.
Then it will add the IP of my dynamic DNS host that is updated by my home router.

I tested out the script to make sure it worked. After running it, I verified the table value.

```
pfctl -t allowssh -T show
```

That command should list the current IP.

Now that I have a working script, the next step was to have it run on boot. This can be done on OpenBSD with `/etc/rc.local`, which will during the init process after pf and dns are running so the table can be populated.

I edited it to run my script.

```
/home/myuser/scripts/pf_allowssh_update.sh
```

With that, I'll allways be able to ssh in after a reboot. With that taken care of, I need to run the script on an interval to catch anytime my IP address updates. I setup a `cron` job to run the script every five minutes.

On OpenBSD, if you do not already have a crontab, you can set one up with the following steps.

```
doas su  #gain root
touch /etc/crontab
chown root:wheel /etc/crontab
chmod 0600 /etc/crontab
```

Populate the crontab with the following line to run the script [every 5 minutes](https://crontab.guru/#*/5_*_*_*_*).

```
*/5 * * * * root /home/myuser/scripts/pf_allowssh_update.sh
```

The final step was testing it out. I flushed the table manually and then existed my ssh session.

```
pfctl -t allowssh -T flush
```

At that point, I tried to ssh in and was denied. I waited 5 minutes and tried again and I was allowed in.

For fun, I verified that the table once again showed my dynamic IP

```
pfctl -t allowssh -T show
```

I could also see the cron run by checking the log

```
tail -f /var/cron/log
```

That's about all. As always, be super careful when messing with SSH and firewalls on a publicly facing server.

### Allowing rsync deploys

I also need to allow `rsync` deploys from the CI/CD pipeline.  An API is available to get runner IP ranges which I also setup a table to contain.  This time it was a file persisted table.  Using `curl` I query the api and then use `jq` to write the array of IP ranges to a file that matches the persistent rule.  Finally, I use `pfctl` to `replace` the current table from the file.

This is in a daily cronjob as the IP API is updated weekly and should be sufficient.

References:
- https://std.rocks/openbsd_crontab.html
- https://www.openbsd.org/faq/pf/tables.html
- https://man.openbsd.org/pfctl
