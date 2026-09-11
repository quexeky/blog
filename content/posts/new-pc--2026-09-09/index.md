---
title: New PC because compiling fast is fun
date: 2026-09-09T14:04:00+10:00
description: Thinking rocks go brrrrr
cover:
  relative: true
showToc: true
---
# Hardware

I have FINALLY (mostly) built a new computer for all of my programming and other such hobby-ing needs. The only thing left is a brand new GPU, but, well, considering that the 9070XT has gone up $300AUD ine about a month, I'm not feeling particularly inclined to buy one right away. 

Specs:

* Intel Core Ultra 270K Plus (terrible naming scheme but oh well. The 'Plus' is just the cherry on top)
* 32GB Crucial Pro OC DDR5, 6400MHz, CL38
* Gigabyte Z890 EAGLE WiFi 7 Plus (I think that motherboards are significantly worse than CPUs when it comes to naming)
* rm850x PSU
* Fractal Design North (this thing is absolutely gorgeous - oh man)
* Noctua NH-D15
* 1660 Super

Plus whatever storage I could scavenge from various computers, including a 500GB NVME, 500GB 2.5" SSD, and a 4TB IronWolf that I bought a few years ago and never really put to use.

This has been far more of a journey than it has any right to be - not that it was especially worse than I had expected (aside from the indecision in actually choosing the parts I wanted ;-;).

The biggest problem that I had was buying RAM (because of course it was a problem in this economy). I originally bought a pair of 16GB Crucial Pro OC DDR5, 6400MHz CL**32** sticks from Amazon, but whether due to bad luck, bad timing, or a combination of the two, my package arrived completely empty aside from the receipt from Amazon Germany. Not happy at all.

Still, at least there were things to take from it - as a result, I had to get my first Statutory Declaration signed by a Justice of Peace (just saying that "yes I declare that I didn't get this package and you can arrest me if I didn't"), which was something that I'd never done before. I had a lovely discussion with the JP who witnessed it, and he pointed out that it's an excellent way to meet new people, which is something that I'm looking to do at the moment (anyone who wants to just chat or ramble about literally anything hmu on twitter, linkedin, or even email me IDM - see links at the bottom), since PEOPLE ARE SO INTERESTING.

Anyway, Amazon sent me my money back, and I immediately went and ordered slightly worse RAM for roughly the same price off Scorptec, because I have realised that local businesses tend to do things far better than megacorporations, at least when it comes to selling physical *things*. As such, almost my entire PC was bought off Scorptec; the only thing remaining is the GPU (which I will also probably buy from them), and the PSU. Excellent experience will shop again.

Speaking of PSUs, my other major pain point was just that (albeit, it wasn't really that much of one). I made the mistake of judging PSUs by the 80+ rating, assuming that a Platinum PSU would be better than a Gold one in terms of reliability, which isn't necessarily the case. Rather, since PSU ratings truly only do account for power efficiency, the better way to go about it is just to find a tier list (I used [SPL's Tier List](https://psutierlist.org/)), filter by the specs that you need, sort by rating, and choose the highest rated one that's in your budget. 

Anyway, I *didn't* do this initially, and instead spent far too long looking around for PSUs, finding the Seasonic FOCUS PX, which does seem to be a very solid PSU, but I didn't really need that much efficiency. As such, after I ordered it from MWave, read enough reviews to make me second guess about MWave and their customer service, took a moment, and then looked around for other PSU options (the FOCUS PX wasn't available on Scorptec), I settled on the rm850x by pretty much doing what I said before, then reading a few reviews to make sure that everything would be all good, and getting it hot off the Amazon press as my last order from Amazon for a while (I'm still very pissed about the whole RAM situation). 

Lesson? Getting a good PSU is a far simpler task than I initially thought: reliability is the main thing that you want out of it, but reliability and rating don't correspond like I thought. There is probably something to choosing the **perfect** power supply, but "good enough" is where it's at for me, personally. In any case this PSU will probably outlast my PC unless I manage to zap it hard enough. And even then it'll be the PSU that blows rather than my PC, so in that worst case I'm probably "saving" (aka just not spending) a decent chunk of money by not needing to replace every component.

# Software Setup

This will be a shorter section because I'm planning on writing another post about my actual infrastructure, but the gist is that I want to be able to use this PC as both, well, a PC, but also as a server. As such, I needed things to mostly just work as much as possible out of the box (or with very little configuration), and so my choice of Linux distro (because there's no way I'm running a Windows server) was Fedora. 

My reasoning is as such: I want a well supported distro, which means using one of the most popular ones: Ubuntu, Debian, Arch, Fedora, Mint, and Nix  (I'm going to leave out MX Linux, Zorin, and AnduinOS which are apparently also popular but I've never heard of them so [](https://www.theatlantic.com/technology/archive/2014/05/the-best-way-to-type-__/371351/)¯\\_(ツ)\_/¯). For the sake of conciseness I'll leave out some other favourites, namely Bazzite, Cachy, Pop!, and other such distros, since I haven't used them before so can't really say much about them).

On my last desktop, I was running NixOS for about a year beforehand, and while I appreciate a fully declarative OS as much as the next nerd, it was simply too much to deal with, all the constant recompiling and rebuilding and reorganising my dotfiles. I think I had about 30 separate files the end of it? (I treated it too much like an actual software repo) In any case, Nix was immediately off the table. 

Debian was tempting, but the last time I used it, I kept finding that I needed features that weren't included in the current Debian repos, which was frustrating to get around. No thanks. 

Ubuntu, personally, I would avoid just for the sake of not dealing with Snap. I hate the damn thing.

Arch is what I use on my laptop (btw), but the instability of rolling release keeps adding up until I want to do terrible things to any device that uses it after long enough. I'm surprised that my laptop has survived this long on a single installation, to be quite honest.

Mint is, well, Mint. I've used it a few times when setting up an older computer, but I don't really have enough experience to give much more than a basic explanation that "it kinda just does things and I don't worry about it" - I've found it to be a nice experience, but it's less geared towards workstations and power users than I'd like for these purposes.

Finally Fedora, which ironically I haven't actually personally installed on my computers, but @DecDuck was running it for a while on his Mac, and any distro that can maintain relative stability on an M1 (or M2 or M3? I forget which one he had) is probably a solid choice. That, plus the more security geared features made it tempting enough to try, and I must admit, so far, it's amazing.

Finally I decided to give it a spin to test how it would run and oh wow. Compiling the [Drop App](https://github.com/Drop-OSS/drop) in release went from a little under 10 minutes to a little under 2.

I don't think I can effectively explain how nice this is. I've spent hours just watching rust tick away at its next compile step. Cutting that by a whole five times is like a dream come true.

I'll make a post about my server infrastructure soon. Goodnight for now!
