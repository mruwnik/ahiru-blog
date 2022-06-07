---
title: "MDP"
date: 2022-06-07T13:16:19+02:00
tags: ["Raspberry Pi", "Linux", "Media server", "Devops"]
draft: false
---

I got a new work laptop with various spyware installed. I therefore want to limit its access to my systems, what with me being paranoid. Previously, if I wanted to listen to music, I'd connect my NFS music partition and play it from my work computer. But why go through the intermediary, when I can play stuff directly from my Pi? Especially as it was originally supposed to be a media center.
  Seeing as my Pi is headless, I need an interface of some kind. Preferably one that can be accessed from multiple devices. The simplest option, of course, is some kind of web thingy. Especially as I already have that set up for books and torrents.

# MDP

A quick search found [mdp](https://mpd.readthedocs.io/en/stable/user.html), which seemed to fulfil my purposes. An additional bonus is that it has an [Arch wiki page](https://wiki.archlinux.org/title/Music_Player_Daemon) - that always boosts my confidence in a given program.

## Installation

Installation of mdp is simple enough - `apt install mpd` does the trick. Though in my case I first had to upgrade stuff. Which really should be done a lot more often...

## Configuration

Mpd is configured via `/etc/mpd.conf`. The only thing needed to get things started was to set `music_directory  "/media/data/Music"` in order for `mdp` to know where to find my music. That being said, `mpd` won't actually check for music files etc. - that's up to clients (see below).

## mpd over sockets

Apt installs `mpd` as a service, but configured to use http for communication. To use sockets, the [following](https://mpd.readthedocs.io/en/stable/user.html#systemd-socket-activation) have to be executed:

    sudo systemctl enable mpd.socket
    sudo systemctl start mpd.socket

In my case the second command kept failing until I restarted `mdp` (`sudo systemctl stop mdp`). Which in retrospect is quite obvious...

## Volume control

The default seems to be to use alsa for playback. Which on my Pi worked fine. Until I wanted to change the volume via a client. It took me a while to work out what values were needed, but this [post](https://forums.raspberrypi.com/viewtopic.php?t=150505) pretty much explains how to set it up. The following worked for me:

```
audio_output {
	type		"alsa"
	name		"My ALSA Device"
	device		"hw:CARD=Headphones,DEV=0"
	mixer_control	"Headphone"
	mixer_index	"0"
}
```

# MPC

Mpd by itself doesn't do all that much. You also need a client. To test that things were working, I went with the basic command line client, which is [mpc](https://www.musicpd.org/doc/mpc/html/). This can be installed via `apt install mpc`.

Once it's installed, run `mpc update` to make sure mpd scans all music sources. 
To check what's available, run `mpc ls`. This will list the folders in the music library, e.g.:

    The Police
    Akeboshi
    Steph Macleod
    Akurat - Prowincja
    Neil Young - Harvest
    Jack White - Lazaretto
    mutyumu
    Globus - Epicon

To search for a specific artist, run e.g. `mpc search artist akurat`, which in my case resulted in the following:

    Akurat - Prowincja/01 - Akurat - AudioTrack 01.mp3
    Akurat - Prowincja/02 - Akurat - AudioTrack 02.mp3
    Akurat - Prowincja/03 - Akurat - AudioTrack 03.mp3
    Akurat - Prowincja/04 - Akurat - AudioTrack 04.mp3
    ...

Now I executed the following to add a song and start playing it:

    mpc add "Akurat - Prowincja/01 - Akurat - AudioTrack 01.mp3"
    mpc play

So much for testing it out.

# RompЯ

The next step was to install a web client. The mpd website has a [list of http clients](https://www.musicpd.org/clients/), so I just had a look through them and picked whichever had the nicest photos, the winner being [RompЯ](https://fatg3erman.github.io/RompR/Installation).

## Installation

RompЯ is a PHP website, so all that is needed to install it is to download and unpack a [zip file](https://github.com/fatg3erman/RompR/releases) to wherever it should be served from. Seeing that I already have a `media` website set up, I decided to have it there, so it went to `/var/www/media/music`.
Before RompЯ will work, it needs to check its settings. Which are written to the server in the `prefs` folder. This doesn't exist (I'm guessing because git doesn't send empty folders and they didn't want to have extra files in there), so it needs to be created. Ditto if custom album pictures are to be uploaded:

    mkdir music/prefs
    mkdir music/albumart
    chown -R www-data:www-data music/

## Nginx

I use Nginx as my server, which has detailed instructions on how to setup on the [RompЯ website](https://fatg3erman.github.io/RompR/Recommended-Installation-on-Linux). This pretty much boils down to:

### Install needed plugins

    apt-get install nginx php-curl php-json php-xml php-mbstring php-sqlite3 php-gd php-fpm php-intl imagemagick

### Add entry to nginx config file

In my case this is `/etc/nginx/sites-enabled/media`

    location /music/ {
      allow all;
      index index.php;
      location ~ \.php {
        try_files $uri index.php =404;

        fastcgi_pass unix:/run/php/php7.3-fpm.sock;

        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $request_filename;
        include /etc/nginx/fastcgi_params;
        fastcgi_read_timeout 1800;
      }
      error_page 404 = /rompr/404.php;
      try_files $uri $uri/ =404;
      location ~ /albumart/* {
        expires -1s;
      }

      # Use http auth
      auth_basic "Music";
      auth_basic_user_file /etc/apache2/.htpasswd; 
    }

The only differences from that on the website are changing the endpoint name ('/rompr/' -> '/music/') and adding http auth.

### Update php limits

This is done in `/etc/php/7.3/fpm/php.ini` (or the appropriate for whatever version is used). The following values should be set:

    allow_url_fopen = On
    memory_limit = 128M
    max_execution_time = 1800

# Testing time!

To make sure everything was working, I restarted the services:

    systemctl restart mpd
    systemctl restart nginx

This shouldn't have been needed, as I was restarting them along the way, but better safe than sorry.

The only thing left was to check whether everything worked, by going to the [/music/](https://media.ahiru.pl/music/) endpoint and inputting my user and password.
That's it. Now I can control my music from any browser, anywhere in the world. Not that it makes all that much sense when I'm out of hearing of the loudspeakers...

# Updater

One thing that seemed useful, and that isn't automatically added, is an updater of the music library. This can be done manually via the various clients, but seems onerous. So a simple cron task in `/etc/cron.daily/music-update` will do the job:

     #!/bin/sh
     #
     # Rescan the mpd music library to index new files 

     /usr/bin/mpc rescan --wait
