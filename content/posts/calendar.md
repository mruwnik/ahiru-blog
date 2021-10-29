---
title: "Radicale"
date: 2021-10-29T14:20:09+02:00
tags: ["Raspberry Pi", "Linux", "Media server", "Devops"]
draft: false
---

A large part of the reasoning behind having my own media center was data control and privacy. I started using [Migadu](https://www.migadu.com/) for email a while ago, as I decided I'd had enough of giving away my data to the various free email providers. Yes, I know that unless both sides of exchange take care of privacy then nothing really changes, but it has to start somewhere.
Having weaned myself off of Googles mail and storage systems, there was still one killer app left to migrate, i.e. Google Calendar.

Since I started this knowing virtually nothing about calendar systems, my first step was education. A quick google (hehe) later, I ended up with deciding on [Radicale](https://radicale.org/3.0.html#tutorials/versioning-with-git) as the Server and [InfCloud](https://www.inf-it.com/open-source/clients/infcloud/) to have an additional web interface. InfCloud isn't really needed - both thunderbird and KOrganizer handle CalDAV perfectly well, but it'll be nice to be able to access my calendar from other peoples computers. The following description is by no means chronological - I had a lot of problems with various components, most of which stemmed from my ignorance, so for clarity I'll skip the problems.

# Radicale

The [Radicale docs](https://radicale.org/3.0.html#getting-started) are really quite well written and easy to follow. The only thing I'd modify would be to recommend to install Radicale in a virtual env, though that really is out of the scope of a simple getting started article.

The first thing to be done was to add a radicale user. Defense in depth an all.

    useradd --system --user-group --home-dir /media/data/calendar --shell /sbin/nologin radicale

The home dir is `/media/data/calendar` since I want to keep data out of the root partition. In this case it's more fuzzy than in the case of films or music, but it's my party, so there.

Next was the virtualenv with `python3 -m venv /media/data/calendar/radicale`. Virtualenvs are a heavenly blessing and should be used a lot more often.

As I can't be bothered to log into the Pi everytime Radicale dies, I set it up as a service according to the [docs](https://radicale.org/3.0.html#tutorials/running-as-a-service). the on;y thing changed were the following lines:

    ExecStart=/media/data/calendar/radicale/bin/python3 -m radicale
    ReadWritePaths=/media/data/calendar/

I want to access my calendars from the internet via the `media.ahiru.pl` domain, so nginx also needed an entry, also per the [docs](https://radicale.org/3.0.html#tutorials/reverse-proxy).

Authentication is via basic authentication. Everything goes over https, so I assuming that should suffice for now. Seeing as I already had that configured for the ruTorrent, it was simple to reuse its htaccess file.

The last thing was to add git, also described in the [docs](https://radicale.org/3.0.html#tutorials/versioning-with-git). Cause who doesn't want to be able to recover deleted data? I initialised the git repo in the `collection-root` folder, while the docs seem to assume that it will be done in the base dir (whatever `ReadWritePaths` points to). This cause problems, with the git hook (obviously) not working. So I just added a `cd` to the hook command in the `storage` section.

The resulting config file goes to `~/.config/radicale/config`:

    [server]
    hosts = 0.0.0.0:5232, [::]:5232

    [storage]
    filesystem_folder = /media/data/calendar/
    hook=cd /media/data/calendar/collection-root && git add -A && (git diff --cached --quiet || git commit -m "Changes by %(user)s")

    [auth]
    type = htpasswd
    htpasswd_filename = /etc/apache2/.htpasswd

And that's all that's needed to get a CalDAV server up and running. I tested it out with thunderbird and KOrganizer and everything worked fine, which is always a nice surprise.

# InfCloud

For a web interface I went with [InfCloud](https://www.inf-it.com/open-source/clients/infcloud/), mainly because it was the first working thing I found... It's simple JavaScript and HTML so no special installation is needed other than it being unpacked in the server files dir (in my case `/var/www/media/`). That being said, it still has to be configured, which is done in `config.js`. I initially went with the `globalAccountSettings` mode with hardcoded credentials just to check if it works, but after displaying my calendars I quickly changed to `globalNetworkCheckSettings` - no way am I going to leave hardcoded credentials anywhere accessable. I had a problem with the settings not being updated, but that mostly stems from not reading the docs first - they say that `cache_update.sh` must be executed for any changes to take effect. In my defense (not really, but nothing like deflection), the script didn't work for me and I had to manually change the version in `cache.manifest` before the changes were seen. Other than that it also just worked™.

# Clients

That's pretty much all needed to set up a calendar service. Most mature clients support CalDAV, so all that was left was to connect devices etc. to the server. Thunderbird and KOrganizer worked right away (once I found out how to connect them). but IOS didn't seem to be able to manage it. Radicale is available at `https://media.ahiru.pl/radicale` and was inputted as such into the IOS form, but IOS kept removing the `/radicale` path. After lots of tries it suddenly started working, but only after it added the user name to the url (e.g. `https://media.ahiru.pl/radicale/harry_potter`). I don't know who to blame for this, so I'll chalk it up as a mutual misunderstanding?
