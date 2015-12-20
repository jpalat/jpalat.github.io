---
layout: post
title: Deploying a Go app in Debian for Ubuntu
---

I wrote a HipChat plugin in go based on [this article](https://developer.atlassian.com/blog/2015/09/easy-hipchat-addons-in-golang/).  I've had an itch to try go, but haven't had many opportunities.  I figured this would be a good place to start by transforming it to something that meets my needs.  It's been an interesting start, still getting Go syntax down but I'm getting places pretty quick.  One of the features that attract me to go is that I end up with a binary I can run on an eqivalent system pretty easily.  This article doesn't address go, but does talk about how I packaged this application for future use.

To Build the bot:
Running 'go install rhiza.com/gotham-hc-buildbot' will create a new binary  $GOPATH/bin/gotham-hc-buildbot

To build the Debian package:
Build Bot layout

    BuildBot
    ├── DEBIAN
    │   ├── conffiles
    │   └── control
    ├── etc
    │   └── init
    │       └── buildbot.conf
    └── opt
        └── buildbot
            ├── bin
            │   └── gotham-hc-buildbot
            └── static
                ├── buildbot-connect.json
                ├── config.hbs
                ├── info@2x.png
                ├── info.png
                └── layout.hbs

The working bits are in /opt/buildbot/bin 
The static bits (images, json) are in /opt/buildbot/static

The upstart files are in etc/init/buildbot 

The DEBIAN directory has the details of what the deb file should do.  The DEBIAN/conffiles is a list of configuration files.  Users will be prompted before the conffile is overwritten.  DEBIAN/control are the details of the package:

```
palat@Interknights:~/Projects/debian$ more BuildBot/DEBIAN/control
Package: buildbot
Version: 0.3
Section: utils
Priority: optional
Architecture: amd64
Essential: no
Depends:
Maintainer: Jay Palat
Description: A bot that talks to Rhiza's build machine to deploy.
 A HipChat bot that talks to Rhiza's build machines to
 deploy code to their infrastructure.
```


By default it will copy the directries to the paths given (/opt/buildbot and /etc/init/buildbot) creating new directories as necessary.




``Run : fakeroot dpkg-deb --build BuildBot``

This creates BuildBot.deb

install with ``dpkg -i BuildBot.deb``

Update the config script with changes.
