---
title: Azure MediaWiki Stack Part 2
date: 2017-05-22
tags: MediaWiki
author: Sam Joseph
---

![mediawiki](/images/MediaWiki.svg)

I wasn't able to complete a re-run of the NHS HLP MediaWiki installation due partly to getting distracted by other work, but also because the Parsoid service (needed by the VisualEditor) wouldn't install.  I was following the instructions in https://www.mediawiki.org/wiki/Parsoid/Setup but getting "E: Unable to locate package parsoid".  I thrashed around, got distracted and then at the end of the day the command just worked.  I had a similar experience with letsencrypt on AsyncVoter, where the letsencrypt server was down, so maybe that was the issue with installing Parsoid, that the wikimedia releases site was down.  Anyway, next step is to configure the Parsoid service, which involved editing `/etc/mediawiki/parsoid/config.yaml` and updating the uri, which defaults to `http://localhost/w/api.php`.  After various trial and error I'd learnt from the previous install that we needed to drop the w, like so:  

```
 # Configure Parsoid to point to your MediaWiki instances.
        mwApis:
        - # This is the only required parameter,
          # the URL of you MediaWiki API endpoint.
          uri: 'http://hlp-wiki-develop.agileventures.org/api.php'
```

Noting that we need to use `sudo` to make adjustments to this file.  Then we're supposed to adjust settings.js, but I could never find that file and so created it from scratch in the same directory like so:

```js
parsoidConfig.setMwApi({ uri: 'http://hlp-wiki-develop.agileventures.org/api.php', prefix: 'localhost', domain: 'localhost' });
```

The instructions make it look like `prefix` and `domain` need updating, but the above incantation with 'localhost' was the one that worked for me last time around.  Looking at the live system I wonder if the Parsoid service needs to run over the live domain ('wiki.healthylondon.org') or if it could go direct to `bitnami-mediawiki-8a65.cloudapp.net`?  Maybe, maybe not, and could other configuration tweaks make the VisualEditor load a bit faster?

Anyhow, to get the basic VisualEditor up and running we probably need to restart the Parsoid service and add some configuration to the mediawiki instance to know where to talk to the Parsoid service.

```sh
$ sudo service parsoid restart
 * Restarting Parsoid HTTP service parsoid
Started Parsoid server on port 8142
```
brings the Parsoid server back up on port 8142, and then the following in LocalSettings.php connects the two up:

```php
## Parsoid Config for Visual Editor

$wgVirtualRestConfig['modules']['parsoid'] = array(
        // URL to the Parsoid instance
        // Use port 8142 if you use the Debian package
        'url' => 'http://hlp-wiki-develop.agileventures.org:8142',
        // Parsoid "domain", see below (optional)
        'domain' => 'localhost',
        // Parsoid "prefix", see below (optional)
        'prefix' => 'localhost'
);
```

In principle this could all work, but we get errors like this when we try to use the VisualEditor:

![could not find server error](https://www.dropbox.com/s/xr1198z72jforf0/Screenshot%202017-05-22%2010.21.31.png?dl=1)

And I think the issue here is that the client JavaScript in Mediawiki is trying to contact back to the server on port 8142, which isn't open on the Azure box, so we need to make a change there.  This involves adding a "Network Security Group":

![adding a "Network Security Group"](https://www.dropbox.com/s/w9sb7stbq88wams/Screenshot%202017-05-22%2010.26.37.png?dl=1)

and then an "inbound security rule":

![an "inbound security rule"](https://www.dropbox.com/s/59o0krege5temk9/Screenshot%202017-05-22%2010.28.20.png?dl=1)

but then we still get errors like:

![timeout reached](https://www.dropbox.com/s/l810011weec2vef/Screenshot%202017-05-22%2010.34.32.png?dl=1)

in which case the Parsoid troubleshooting documentation is useful:

https://www.mediawiki.org/wiki/Parsoid/Troubleshooting#Configuration

Although I was stuck here for a while, but I think it comes back to the problem that it doesn't involve adding a security group - at least the way I got it to work last time (which took a while to figure out) was to add an extra endpoint rule to the virtual machine.  Although maybe the security group IS needed.  Blurgh, some experimentation needed here, and this whole process of documenting the setup seems to be taking just as long as figuring out the setup in the first place ...!

![adding an endpoint to the virtual machine on Azure](https://www.dropbox.com/s/ghzx10bnhbq7nbm/Screenshot%202017-05-22%2011.04.09.png?dl=1)

Ah, so it looks like we need BOTH the endpoint AND the security group; although I'm not 100% on that, and I think I'm out of time to really determine that ... on to [part 3](http://nonprofits.agileventures.org/2017/05/23/azure-mediawiki-stack-part3/)




