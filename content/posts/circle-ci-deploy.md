---
title: "Circle CI Deploy"
date: 2020-09-06T18:41:22-05:00
categories:
 - ci/cd
tages:
 - circleci
slug: circle-ci-deploy
---

I had already had another site using Circle CI to build and deploy a Hugo site from source on GitHub. I looked at Github Actions but it was a tad confusing so I figured I get the build and deploy going with something I knew. The only difference I really wanted to make was to use rsync instead of lftp. The other site only had FTP or FTPS as an option, so I used lftp to publish the site via FTPS.  However, just able everything else in my CircleCI config would do.

Step 1 was to create a user on the target machine that only had write permissions to the wwwdata folder.  I chose to set this up via a group and used ACLs to configure the permissions so that the original user / group permissions stayed the same.  This took a while to figure out because even though I had given the group recursive read/write permissions, it still didn't have permissions to create new files. TIL that under the ACLs, creating a new file/folder is separate from the write permission.  Looking at the manpage for `setfacl` and comparing the `getfacl` output from a user that could write, I realised the `p` (append_data) permission is needed as well. I'm also using ZFS, so the `:allow` is required.

```
setfacl -R -m g:webauthorsgroup:rwp:allow /path/to/web/wwwdata
```

Once I had that figured out, the next step was to setup an SSH key for this user from the CircleCI pipeline. CircleCI requires keys in PEM format, so I setup a key and saved it in the project secrets. I added the public key to the `authorizedkeys` for the webpublsh user.

I tested the rsync command manually to ensure it worked before adding it to the .circleci/config.yml file and files were sync'd correctly.

I had an issue in that the rsync would fail because my server's key was not in the ephemeral know_hosts file of the build pipeline.  To fix this, I copied the output of `ssh-keyscan` and placed the output in an environment variable that is send to the known_hosts file during the build.

To wrap things up, I set a few variables for the user, host, and publish directory.  Finally after about 30 failed builds, I finally saw some green successful builds.

CircleCI is fine and I appreciate free, but the documentation, while it exists, does not seem to cover the basics. In reading over the docs and examples it seems like it's hit or miss whether env variables need to be enclosed in curly braces.  There are examples with $CIRCLE_ENV_VAR and ${CIRCLE_ENV_VAR}. I suspect this is probably actually dependent on the image OS / shell, but I found it confusing considering I don't do a ton of shell scripting.  Something to dig into.