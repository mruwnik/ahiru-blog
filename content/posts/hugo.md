---
title: "Hugo"
date: 2021-10-12T16:54:08+02:00
tags: ["Raspberry Pi", "Linux", "Media server", "Devops"]
draft: false
---

I've had the idea of a blog tumbling around my head for a while now. Not that I have anything specific to write about - more just a general thought that it could be good to write my ideas down. For a few years I've been doing the equivalent of an blog via email to a friend. They don't react in anything like realtime (generally replying to a whole bunch of emails every monthish) so it's pretty much like sending my thoughts out into the ether. This way I'll at least have a log of what I've been thinking. In a binary format for ease. So a binary log. Maybe `blog` for short?  Anyhoo. Another potential useful side effect is word retention. I keep forgetting words. I know that they exist, what they mean, how to use them, but can't remember the actual word. While I don't know if that even works, it seems like it's worth a shot.

## Where

While [setting up](/posts/init/) my [media server](https://media.ahiru.pl/) I decided to have 2 virtual servers - one for the media stuff (an explanation of what's what, ruTorrent, etc.) and another as the main [ahiru.pl](https://ahiru.pl) landing page. I could have set it up with the domain register, as they threw in a static file server along with the domain, but I prefer to have control over my stuff. Even if it means that it's more flaky (what with it being on a Raspberry Pi rather than a proper server). So I ended up with the server simply returning Nginx errors.

## What

I didn't want to hand code HTML files. Not that I couldn't, of course, but I'm not a masochist. So I needed a HTML content server type thingy. Wordpress, Joomla etc. are way overkill for a simple blog. WSIWYG editors are nice and all, but tend to be complex. I had no need for users or a whole portal. I also didn't need any javascript magic. Simple HTML and CSS for me, please. What I wanted was a simple an efficient static site generator. Where I'd write posts in markdown or something and generate the HTML from it.

I'd heard about a few static site generators, mainly [Hugo](https://gohugo.io/) and [Jekyll](https://jekyllrb.com/). After some 2 minutes of very quick skimming I decided to give Hugo a shot. It worked fine, which was all I wanted, so for now Hugo it is.

## How

Setting up Hugo took more work that I initially thought. First I tried a simple `apt install hugo`. Which worked fine - the project was set up without any problems and it successfully generated HTML. So then I decided to choose a theme, ending up with [Cactus](https://github.com/monkeyWzr/hugo-theme-cactus). Which is where the problems started. It turns out that the APT version of Hugo is quite old (as they tend to be) and Cactus wouldn't compile without a newer version. No problem, thought I, it should be easy to manually download a newer Hugo version, right? So I went to the [installation page](https://gohugo.io/getting-started/installing/) to find instructions.

Here I should probably stop to remark that before one starts shouting at others, it's probably best to make sure you didn't miss anything. While writing this I decided to verify whether there really wasn't a binary that would work on the Raspberry Pi and it turns out I'm mistaken. [This](https://github.com/gohugoio/hugo/releases/download/v0.88.1/hugo_0.88.1_Linux-ARM.deb) seems to work fine...

Anyway - I took the long way round and first installed golang (again manually as APT was out of date), then compiled Hugo, after which Cactus didn't report any errors.

## When

Manually logging on to the Raspberry Pi each time I wanted to edit anything wasn't a very nice prospect. So it was time for automation. I decided on a git repo which would be pulled and built every 5 minutes. Assuming that the master branch is kept sane, this should be quite a decent and scaleable solution. The only potential problem seemed to be potential conflicts if the master branch was overwritten. Yes, I know that shouldn't happen and if it does then something is wrong, but I knew that would happen at some point. A single problematic `git commit --am` had the potential to break the whole build. The way I solved this was a simple (albeit ugly) force pull of the master branch before building it, using the following commands:

    git fetch origin master
    git reset --hard FETCH_HEAD
    git clean -df
    /usr/local/go/bin/hugo -D

## Why

Coz why not :D
