---
title: My Hugo + Papermod, Umami, and Cloudflare setup
date: 2025-12-16T20:20:00+11:00
draft: true
tags:
  - how-to
categories:
  - Technical
description: A documentation of all of how I set up my personal blog, complete
  with all of the bells and whistles
---
## Why not Ghost?

Alright here's the scenario: I have almost nothing to do at the moment. I've finished my final exams, don't currently have a job, but want / need some sort of creative outlet.

And then it comes to me - a blog! They don't seem too hard to set up; I've seen one of my friends set up Ghost before, and all it took was a docker compose, right?

I was then shot 57 times.

My problems with Ghost are such:

1. SMTP is **required**. I'm planning on setting up a newsletter at some point (not like anyone actually cares but yk, it'd be cool), but I don't want to have to go through all of that setup (I'm looking at you, smtp.mailgun.com vs smtp.mailgun.org) and config before I get to make a single post. It destroys my inspiration
2. I personally really want to self-host as much as possible. While yes, Ghost supports it, there are some major issues that I have, the first being that my router is behind CGNAT, which means Cloudflare tunnels and needing to set up nginx configs. Again, while entirely possible, I am lazy. 
3. The other disadvantage to self hosting is that while Ghost isn't heavy, it's certainly not light. That, in addition to ideally a self-hosted tinybird instance (which for my initial run I didn't actually do - I just used the official hosting) just made it a little too much for me to stomach with my little Pentium G4400 and 8GB of DDR3 RAM.

So I decided to pivot, and after a little bit of googling, the name that kept popping up over and over again was Hugo, the customisable static site generator that promised to be the end to all of my problems.

## The actual setup

I'm going to split this up into different sections so that you don't actually have use all of them, because I mean, it's not like they actually interact. The only one that's really necessary is Hugo + PaperMod for the specific configurations, but most themes have their own ways to add headers and partials, which is really all that you need.

### Hugo + PaperMod

For a local instance of Hugo (I'm going to omit the "+ PaperMod" now), you'll first need to install it. Personally, I use NixOS, and it was as simple as adding:

```lua
environment.systemPackages = with pkgs; [\
  hugo\
];
```

To anywhere within my system config (I might make a post about this later)

For other systems, you can find how to install Hugo on their [website](https://gohugo.io/installation/).

Next, it's as simple as 

```shell
hugo new site MyNewBLog --format yaml
cd MyNewBlog
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
git submodule update --remote --merge # For if you want to update the theme
```

Then, in your text editor of choice, open up the `hugo.yaml` file and add the following:

```yaml {hl_lines=["4"]}
baseURL: https://example.org/
languageCode: en-us
title: My New Hugo Site
theme: [ "PaperMod" ]
```
