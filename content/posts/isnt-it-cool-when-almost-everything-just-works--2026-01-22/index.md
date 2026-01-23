---
title: Isn't it cool when (almost) everything just works?
date: 2026-01-22T19:11:00+11:00
description: My very surprising experience working on Downpour, the CLI tool for
  Drop-OSS Depots
draft: true
cover:
  relative: true
  image: drop.svg
  alt: Drop-OSS Logo
  caption: Drop-OSS - https://github.com/Drop-OSS/
showToc: true
---
Alright so a bit more about me: I'm the second maintainer of the [Drop-OSS](https://github.com/Drop-OSS/) project. I started working on it when my friend [@DecDuck](https://github.com/DecDuck) decided that he wanted to make a platform similar to Jellyfin, but for games. Kinda like a self-hosted steam alternative. My initial role on the project was to develop the [Drop App](https://github.com/Drop-OSS/drop-app), the native companion app which worked with the server, because he wanted the app itself to be extremely fast, and I knew Rust. I'll fast forward a few months from here though, because development of v0.1.0 and v0.2.0 went very, very well. I actually ended up getting a free 3D Printer out of it due to an organisation known as [HackClub](https://hackclub.com/), so I wasn't exactly complaining about the massive amount of effort put into it. You never realise just how many hours of pouring over text on a screen really goes into developing even remotely in-depth projects, setting aside the fact that I was still very much a rookie at Rust, so everything took about twice as long to make. 

But yes, development. To summarise the first few months of work, we:

* Learnt how to use Tauri (WebKitGTK has been, and will always be, a mistake)
* Built the Download Manager, a thing that I was so happy with when I first made it, but looking back isn't as incredible as I first thought. Still not bad for a first major project though.
* Screwed around with git a bunch. I was using an invalid signing key for a bit of my work, and spent far too many hours figuring out how to amend all of the commits that I'd made with it, because me being far too curious, I decided that it was an effective use of my time
* Figured out that both Dec and I had no idea how to manage a project like this, and then managed it anyway
* Learnt why Mutexes (Mutexs?) exist (multithreading and async. That was a wild ride)
* Implemented UMU-Launcher / Proton support
* Talked to far more people than expected in our [Discord server](https://discord.gg/RJJPjVs3sP)

And then back to school it was. I'd "just" started Year 12 (final year of High School in Australia. For public schools it technically starts in the last term of Year 11, but also it was the start of the year in which I was primarily Year 12 so ¯\\_(ツ)\_/¯[](https://www.theatlantic.com/technology/archive/2014/05/the-best-way-to-type-__/371351/) ). And if I'm being honest, I lost a lot of my motivation here. I kept going fairly strong through to until about July, which was when we were releasing v0.3.0, but I was stretching myself. That, on top of exams, assessments, and the general stress of final year got to me, and I stopped for a while. During that span from January to July, we didn't do too bad though:

* Refactored a bunch of the code because it was a nightmare to deal with and compile
* Added .dropdata files for manifests and game installs
* Offline mode and caching (really it was just caching, and then we realised that we could just keep using cached data if the server wouldn't connect. It really needed / needs improvement)
* And a bunch of bug fixes
* Started actually using PRs (occasionally)
* Got quite a few people contributing, both code and
