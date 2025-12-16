---
title: My Hugo + Papermod, Umami, and Cloudflare setup
date: 2025-12-16T20:20:00+11:00
draft: true
tags:
  - how-to
categories:
  - Technical
description: Don't look please
---
## Why not Ghost?

Alright here's the scenario: I have almost nothing to do at the moment. I've finished my final exams, don't currently have a job, but want / need some sort of creative outlet.

And then it comes to me - a blog! They don't seem too hard to set up; I've seen one of my friends set up Ghost before, and all it took was a docker compose, right?

I was then shot 57 times.

My problems with Ghost are such:

1. SMTP is **required**. I'm planning on setting up a newsletter at some point (not like anyone actually cares but yk, it'd be cool), but I don't want to have to go through all of that setup (I'm looking at you, smtp.mailgun.com vs smtp.mailgun.org) and config before I get to make a single post. It destroys inspiration
2. I need to fully self host it. There are other disadvantages to this, but my issue here is that my router is behind CGNAT, which means Cloudflare tunnels and needing to set up nginx configs. Again, while entirely possible, I am lazy. 
3. Ghost is *heavy*. At least, in comparison to Hugo, which doesn't even need to run on my server.
