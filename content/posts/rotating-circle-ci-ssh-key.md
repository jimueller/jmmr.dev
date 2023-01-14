---
title: "Rotating Circle CI SSH Key"
date: 2023-01-14T13:32:53-06:00
categories:
 - admin 
tags:
 - ssh 
slug: rotate-circle-ci-ssh-key 
draft: true
---

With the Circle CI incident where project secrets may have been accessed,
I needed to rotate the SSH key used to deploy this blog to the web server.

## Generate a Key

Follow [Adding an SSH Key to Circle CI](https://circleci.com/docs/add-ssh-key/) to generate
a key in the correct format.  Circle CI uses PEM format in some cases.

## Copy Private Key to Circle CI secrets

Copy the **private** key to the Circle CI secrets, specifying the hostname, and grab the
fingerprint.

## Add the new key to the job

Update the `.circleci/config.yaml` to add the new key.

```yaml
- add_ssh_keys:
     fingerprints:
       - "a4:1u:cg:99:43:42:0c:e2:zz:bd:03:qf:c5:30:g3:p8"

```

## Add Public Key to Server

Add the public key to the `authorized_keys` file on the server for the
rsync user.
