---
title: "Lighttpd Mod Deflate"
description: Configuring lighttpd mod_deflate to compress files over the wire
date: 2022-10-15T21:01:41-05:00
categories:
 - hosting
tags:
 - lighttpd
slug: lighttpd-mod-deflate
draft: false
---

Configuring mod_deflate for lighttpd was pretty simple overall, the main challenge was clearing the compressed file cache after a build.

mod_deflate will compress files over the wire, with the option of caching the compressed files.

The lighttp wiki has all the info on setting up basic compression so I'll just cover the trickier part of clearing the file cache.

The first step is setting the cache directory.

```
deflate.cach-dir = "/var/lighttpd/deflate/cache"
```

Next, create that directory and give the lighttpd user/group read and write premissions.

```sh
mkdir -p /var/lighttpd/deflate/cache
chown wwwuser /var/lighttpd/deflate/cache
```

Reload lighttpd

```sh
service lighttpd reload
```

At this point, you should see files and directories appear in the cache dir.  (You may need to generate some traffic).

Now, we need to determine how and when to clear the cache.  lighttpd docs suggest deleting anything older than 10 days, that would probably work, but since I'm using CI/CD, I can clear the files after a build.

The first step is to give the ci user the appropriate permissions to delete files and folders in the cache dir.  The key to this is to make sure the acls apply to the newly created files, which can be done with the inheritance flag.

```sh
setfacl -Rm g:ciusergroup:D:fd:allow /var/lighttpd/deflate/cache
```

Breaking down the command:

- -m = modify acl
- g = group
- ciusergroup = the name of the CI user's group
- D = allow deleting children
- fd = new files and directories inherit this acl
- allow = allow...

After this, I setup a final step in the CI/CD pipeline to ssh in and delete children of the cache-dir

```sh
ssh user@host rm -r /var/lighttpd/deflate/cache/*
```

Works like a charm.
```