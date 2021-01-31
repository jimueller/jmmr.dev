---
title: "Lighttpd mod_expire"
description: "Configuring lighttpd mod_expire for caching static assets for a small web log"
date: 2020-09-05T18:42:01-05:00
categories:
  - hosting
tags:
  - lighttpd
  - cachcontrol
slug: lighttpd-mod-expire
---

Using mod_expire is pretty easy to set the Expire and Cache-Control response headers. For this site, I'm mainly concerned about caching any CSS or JavaScript, if I use any at all. While the [mod_expire documentation](https://redmine.lighttpd.net/projects/lighttpd/wiki/docs_modexpire) is pretty easy to follow in this case, I'm going through the steps I did, more just to remind myself.

## Enable mod_expire

Being new to lighttpd and having a pretty simple setup, I decided to just keep all configurtion in a `lighttpd.conf` file for now. I have a section near the top to enable modules, so I added mod_expire.

{{<highlight conf>}}
server.modules += ("mod_expire")
{{</highlight>}}

## Configure Expiration

As mentioned earlier, I'm just concerned with the css and javascript so I'm using the `expire.mimetypes` configuration. Once I have more images on this site, I'll probably look at using the `expire.url` config to expire everything in an `/images/` directory.

The syntax for mimetypes is:

```conf
expire.mimetype = ("mime/type" => <access|modification> plus <number> <years|months|days|hours|minutes|seconds>)
```

I'd guess that you could probably put multiple mimetypes in a single line, but I didn't feel like looking up exactly how to do that so I kept each mimetype on a separate line. If I ever do want to configure different expiration settings per mimetype this allows me to do that.

{{<highlight conf>}}
#######################################################################

##

## Caching mod_expire

## --------

##

expire.mimetypes += ("text/css" => "modification plus 30 days")
expire.mimetypes += ("application/javascript" => "modification plus 30 days")

##

#######################################################################
{{</highlight>}}

After configuration, I restarted the lighttpd server and loaded up the site. Upon inspecting the headers for a javascript and stylesheet, I can now see the appropriate headers set.

```
HTTP/1.1 200 OK
Content-Type: application/javascript
ETag: "1717575643"
Last-Modified: Sat, 05 Sep 2020 21:28:23 GMT
Expires: Mon, 05 Oct 2020 21:28:23 GMT
Cache-Control: max-age=2582935
Content-Length: 33912
Date: Sat, 05 Sep 2020 23:59:28 GMT
Server: lighttpd

HTTP/1.1 200 OK
Content-Type: text/css; charset=utf-8
ETag: "840284767"
Last-Modified: Sat, 05 Sep 2020 21:28:19 GMT
Expires: Mon, 05 Oct 2020 21:28:19 GMT
Cache-Control: max-age=2582931
Date: Sat, 05 Sep 2020 23:59:28 GMT
Server: lighttpd
```

That was all there is too it.
