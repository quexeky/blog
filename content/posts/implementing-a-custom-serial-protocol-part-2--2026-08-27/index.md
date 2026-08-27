---
title: Implementing a custom Serial Protocol (part 2)
date: 2026-08-27T19:20:00+10:00
description: More implementation details and exploring various ways to make
  iterators do interesting things
cover:
  relative: true
showToc: true
---
I've come to realise that calling this series "designing a custom serial protocol" is probably a bit of a misnomer, because of the fact that, well, this protocol doesn't necessarily operate over serial. Of course, that's probably going to be the most frequently used method of delivery, considering how easy it is to implement UART, USB CDC, and other such things, but *technically*, this can operate on any existing layer 3 or 4 protocol. 


## Bit by bit (or byte by byte?)

A rather frustrating realisation that I have somehow only just realised is that I'm going to have to process these packets as they come in. Of course, this is the way that you should be doing it. It's just annoying. That being said, it may also be a rather interesting task, as I can probably make an iterator over the entire chunk which takes in a reader and reads enough instructions to make an instruction. Yes, I'll do that.
