---
title: "Sweet berries"
date: 2021-10-09T14:36:09+02:00
tags: ["Raspberry Pi", "Linux", "Media server", "Devops"]
draft: false
---

Seeing as there are those with backups and those that will wish they had backups, I deemed it time to set one up. My ISP gives everyone a public IP and I had a Raspberry Pi 4 lying around from a previous attempt, so I thought that I might as well set up a proper server at the same time. 

## The initial goals:

 * a decent amount of space available (at least a few TB for starters)
 * a media server or something from which films, music etc. could be downloaded
 * a VPN
 * Pihole or similar to get rid of as many adverts as possible
 * public access so I could access my files while away from home

## Router

My ISP gives out a public IP which is very nice, but they only provide the [signal](https://www.youtube.com/watch?v=n1tXlpRq-jY), which means that each customer has to get their own router. They recommend the TP-Link Archer C1200, so I went and bought it. From a quick look over, it looked like it provided a simple way to set up a VPN and it also comes with a USB port to plug in a disk for network access. The Pihole was more of a nice to have, rather than a hard requirement, so it looked like the router would pretty much handle everything by itself. Full of optimism, I ordered a 5TB USB3 HDD, formatted it as NTFS (since it of course only supported windows files) and logged in to the router admin panel to check that everything worked. And of course it didn't. It turns out that there is a known problem where it can only handle up to 2TB of storage. So that was out.

## OMV 

Luckily I had a spare Raspberry Pi 4 lying around from a previous attempt at a media server. First I had a quick check for ready Raspberry Pi distros or projects that do that kind of things. A few years ago I had a setup with [Kodi](https://kodi.tv/) so I was looking for something in the same vein but as a NFS type thingy. [OpenMediaVault](https://www.openmediavault.org/) seemed to be quite popular so I went with that. The [installatio](https://www.openmediavault.org/) n is quite simple on the default [Raspberry Pi OS](https://www.openmediavault.org/), though it's done by curling a bash script, which is always a bit scary. After it was installed, it can be accessed via its web gui (by default exposed on port 80). OMV is pretty much a web GUI for basic Linux user and files management plus configurators for NFS, Samba, FTP etc. I didn't want OMV to be public, so I started from changing the port to 8000.

## USB disk

Raspberry PIs boot from an SD card, which is problematic for speed, size and stamina reasons. Especially as there was a 5TB USB HDD attached. The first thing to do was to get it to use the hard drive for root. This can be done via firmware which means that no SD card is needed, but I prefered to do it the [old fashioned way](https://raspberry-valley.azurewebsites.net/Transferring-System-to-USB/), just in case. This requires the cards root partition to be rsynced to the USB HDD, a change in `/etc/fstab` for it to mount the USB HDD as `/` rather than the SD card and to change `/boot/cmdline.txt`. 

A slight problem with this is that OMV manages `/etc/fstab` and will overwrite changes there. This can be handled by adding an entry to the [OMV](https://openmediavault.readthedocs.io/en/5.x/administration/storage/filesystems.html) `config.xml` file.

The USB HDD was partitioned into a 30GB `/` and 4.5TB as `/media/data` so that if the system needed to be cleaned it wouldnt effect any backups, media etc. Seeing as the users should write to `/media/data` anyway, I decided that users should have home there rather than at `/home/`.

## File shares

OMV sets up various shares, which are folders that can be shared. These shares can then be made available via network file protocols. Which makes controlling sharing quite easy, albeit at the cost of fewer options (as only a few options are presented in the GUI). More complicated settings can always be done via the normal configuration files. I decided that I wanted to have publicly availble Samba shares and NFS set up for the local network. Initially FTP was also enabled, but I decided against it. The following shares were used:
 * Films
 * Serials
 * Anime
 * Music
 * Unsorted

`Unsorted` is a staging area for various media which haven't been sorted into the appropriate categories.

## NFS

 NFS was totally new for me, so it caused a few headaches with permissions - reading the folders worked out of the box, but getting them to be writable took a bit of work. First I tried to muck about with `chmod`, but that didn't help much - I could only write when the folders where globally writeable, which I most certainly didn't want. It turned out that OMV handles the permissions itself and that there's an ACL settings popup that is used to configure each shared folder which allows to set fine grained permissions on the user and group level.

Another problem was with user mapping. The default behaviour is to map users on the basis of thier UIDs. This didn't work too well, as my users had different UIDs on different machines. There is a mechanism for mapping on the basis of user names (idmapping), which I managed to get to work for a moment, but after rebooting it stopped working. At this point I couldn't be bothered to continue messing about with it and just kept a mental map of which raspberry pi users had the same UIDs on which machine. This was enough to get write access on my main machine and read access on the other ones, which was what I wanted in the first place. So now I can seamlessly listen to my music on all my machines. Hurah!

## DNLA

OMV has a plugin for [DNLA](https://en.wikipedia.org/wiki/Digital_Living_Network_Alliance) which I decided to enable. Why not let devices that handle it play music from the Raspberry? This was a simple matter of installing the plugin from the list of available plugins, enabling it and choosing the shares to provide.

## Books

I read a decent amount of books on my [Kobo](https://www.openmediavault.org/), and so have accumlated quite a lot of them. I manage my library with [Calibre](https://www.openmediavault.org/), which has an inbuilt server which can be used to browse, download and even add or edit books. Therefore I obviously wanted to have this on my Raspberry. It can be downloaded via apt, but the APT version tends to be quite old. Which meant a manual download. Which is via a curl to bash.

**\<rant\>** What is it with everyone installing via curl? Yes, it's a lot easier to maintain a single script rather than builds for multiple distros, so I don't blame the authors, but it doesn't inspire trust in me as a user. Though since I trust them enough to install their program, I suppose I might as well trust the install script... But it's a bad habit to get into. And unfortunatly most of the programs used for this project are supposed to be installed like this... **\</rant\>**

Seeing as Calibre was running as a server and so would be receiving public requests, I wanted it to be run as its own user for security. So I added a new user with a home at `/media/data/Books` and copied my current Calibre library there. This meant that no extra imports etc. were needed. 

The calibre server can have users to manage access to the books. I wanted the server to be public so friends and family could access it. But I didn't want them to modify stuff. Which meant that I needed at least two users - a guest user with read only access, and a rw user for myself (and people I trust more). The calibre server has an option for this: `calibre-server --manage-users`. This should be run as the user which will run the server, and in the library directory, as it will write any changes to a local Sqlite database. The command works by choosing options by number, which seems a bit unfortunate - it hinders any automatation of this. Another problem with it is that it allows displaying the passwords of users. Which is a big security issue and suggests that the passwords are stored in plain text. This at least meant that I wouldn't reuse passwords here, which I suppose is a case of *it's a feature not a bug*?

After the users were added and the server started, the books were available through the browser. Unfortunately for my use case of a single guest user for everyone, users can change their password though the web interface. A single joker could block access for everyone. A quick `whereis calibre` allowed me to find `/usr/share/calibre/content-server/index-generated.html` which contained the html for the server. Inside this file was a line which displayed the user config button which I commented out, and voilà - no user settings available. Of course, someone could manually send requests to the server, but if someone with the username and password starts to do that, then I reckon they deserve any havoc they create.

## Book service

Since I can't be bothered to start the Calibre server after every error or reboot, I added a systemd `/etc/systemd/system/calibre-server.service` service to handle it:

    [Unit]
    Description=calibre content server
    After=network.target

    [Service]
    Type=simple
    User=calibre
    Group=calibre
    ExecStart=/usr/bin/calibre-server \
      --port=8123 \
      --disable-local-write \
      --enable-auth \
      --auth-mode basic \
      --max-request-body-size 20 \
      /media/data/Books/

    [Install]
    WantedBy=multi-user.target

## Torrents

Since I have an always on, internet connected device, why not also set it up as a torrent server? For this I chose [rTorrent](https://rakshasa.github.io/rtorrent/), mainly because I already knew it would work as a headless bittorrent client. Since it's popular, the apt version is quite up to date, so I used that. I also added a new `torrent` user, since this was going to be another internet connected service. The defaults are quite decent, so I left them as they are and set up the `torrent` users home as `/media/data/Unsorted`, to keep all downloaded stuff in the staging directory until they got sorted. I added a `/media/data/Unsorted/.watch/start/` directory as a place to add torrent files to be automatically downloaded - this is a very versatile option considering that the directory is shared via NFS and Samba. The only thing left was to add a Systemd service at `/etc/systemd/system/rtorrent.service`:

    [Unit]
    Description=rTorrent
    After=network.target

    [Service]
    Type=forking
    KillMode=none
    User=torrents
    ExecStartPre=/usr/bin/bash -c "if test -e %h/.session/rtorrent.lock && test -z `pidof rtorrent`; then rm -f %h/.session/rtorrent.lock; fi"
    ExecStart=/usr/bin/screen -dmfa -S torrents /usr/bin/rtorrent
    ExecStop=/usr/bin/bash -c "test `pidof rtorrent` && killall -w -s 2 /usr/bin/rtorrent"
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target

## ruTorrent

Rather than log in to the Raspberry every time I wanted to check on how rTorrent was doing, I decided to set up [ruTorrent](https://github.com/Novik/ruTorrent) as a web GUI for it. As always, [wiki.archlinux.org](https://wiki.archlinux.org/title/RTorrent/RuTorrent#Nginx) came to the rescue with instructions on how to do it. Though in my case there was a bit of a problem with permissions - rTorrent initially create the unix socket at `/media/data/Unsorted/.session/rpc.socket`, but php or nginx couldn't access it - nginx kept logging `connect() to unix:/media/data/Unsorted/.session/rpc.socket failed (13: Permission denied) while connecting to upstream`. This I "solved" by creating the socket as `/tmp/rtorrent-rpc.socket` which got things working, but seems like a potential security issue.

## Anime downloader

I watch quite a lot of anime and tend to be up to date with what is released on TV. This means that I have a new episode or two every day. It was a [bit of a bother](https://xkcd.com/1319/) to manually download them each time, so I [wrote a script to do it](/stuff/anime-downloader.py) for me. The quality is, as usual, very much subpar, but it does the job well enough. The basic idea is that there is a base directory with:
* an`.anime_history` file with the paths of anime already watched
* various anime episode files
* a stalled directory containing episodes that haven't been watched yet, but which also don't look like they will be watched in the near future

Each time the script is run, it collects the names of all anime episodes found, parses them to extract the sub group, series, episode number and video quality and then checks [nyaa.si](https://nyaa.si/) to see if there are any newer episodes. If so, the appropriate torrent file get downloaded to `.watch/start` which is then picked up by rTorrent to be downloaded.

New episodes aren't uploaded all that often, and it doesn't really matter if I watch something a few hours, or even days, later than it's published, so the script is executed every 6 hours. This can done via a simple crontab task, but also via the OMV web GUI, which will create a crontab entry behind the scenes. It took a while to get this to work - first I suspected bugs in the script, even though it appeared to work flawlessly. It turned out that the cron systemd service wasn't enabled. This was a bit of a surprise.

## Pi-hole

I abhor adverts. The first thing I do on a new system is to install [uBlock Origin](https://github.com/gorhill/uBlock) to get rid of them. So [Pi-hole](https://github.com/pi-hole/pi-hole/#one-step-automated-install) is a no brainer. It automatically installs a systemd service and creates a user, which means that part can be skipped. Seeing as it's a DNS service, though, it's useless if other machines don't use it as one. My router uses DHCP, which means that all that needs to be done is to set the Raspberry IP as the primary DNS server. I left the default `192.168.0.1` as a secondary DNS address in case the Raspberry goes down (which it will some day, says Murphy). Before that could be done, though, the Raspberry IP address had to be reserved for it - there'd be no point in setting its IP address as the DNS server if the address would be changed by DHCP.

Pi-hole also gives nifty web GUI to manage it and show statistics. 

## Nginx

Since there are a lot of web services started and I want to access them as website, a web server is needed. I like Nginx, so that's what was used. OMV and Pi-hole add their own server definitions, so all that was needed was to set the appropriate ports. Seeing as I didn't want them to be publicly accessible, they got ports over 8000 and that was that. 

For the other services, I wanted the following structure:
* [/index.html](https://media.ahiru.pl/) with a basic description of what's available
* [/torrents](https://media.ahiru.pl/torrents) for the ruTorrent GUI
* [/books](https://media.ahiru.pl/books) for my books

The books endpoint was already protected by Calibre, but the ruTorrent page leaves that to the server itself. Which was solved by a simple htpaccess file. This of course requires https for any real semblance of security, but seeing as that was the plan anyway, it should suffice.
The server definition is as below:

    server {

            root /var/www/media;

            index index.html;

            server_name media.ahiru.pl;

            location / {
                    # First attempt to serve request as file, then
                    # as directory, then fall back to displaying a 404.
                    try_files $uri $uri/ =404;
            }

            location /books {
                    proxy_pass http://127.0.0.1:8123/;
            }
            # pass PHP scripts to FastCGI server
            #
            location ~ \.php$ {
                    include snippets/fastcgi-php.conf;
                    fastcgi_pass unix:/run/php/php7.3-fpm.sock;
            }

            location /torrents {
                    auth_basic "Torrents";
                    auth_basic_user_file /etc/apache2/.htpasswd; 
            }
    }

## https

HTTPS should pretty much always be enabled. Computers are easily powerful enough to handle it, and [LetsEncrypt](https://letsencrypt.org/) handles the certificate. To automatically handle certificate renewal, I used [certbot](https://certbot.eff.org/instructions). Certbot can also add the appropriate entires to the Nginx server definitions.

## Port forwarding

The last thing to do was to forward the 443 (Samba), 443 (https) and 80 (http) ports on the router to the Raspberry, update the [index.html](https://media.ahiru.pl/) file to add a description of what is what, and the media server is set up.
