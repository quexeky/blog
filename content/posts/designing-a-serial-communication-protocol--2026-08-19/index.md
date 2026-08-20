---
title: Designing a Serial communication protocol
date: 2026-08-19T21:13:00+10:00
description: Developing a custom protocol over Serial
draft: true
cover:
  relative: true
  image: protocol.jpg
  alt: Current protocol implementation
showToc: true
---
I've been working on a hardware project with some smart actuators recently, and realised just how much I despise log statements for debugging. They're slow, often require a recompile to change what you're logging, and while easy to implement initially, can become far more frustrating than a reasonable setup.

Let's first establish what I'm looking for:

I want a protocol that lets me connect to a device over a serial stream to rapidly gather logs, telemetry, and device state, *without* interrupting the device. 

As a secondary goal, I'd like to also be able to enable some control mechanism to manually control the device, both in toggling live functions, and also changing settings.

My current scenario is that I want to connect a few smart actuators together into a robotic arm and do some stress tests, and if something goes wrong, I want to just plug my computer in to the actuator that isn't happy, and immediately view all of its logs, as well as any other telemetry as it comes in, rerun the tests, and see what happens. Wouldn't that just be wonderful?

Now, as with any engineering problem, it's worth looking at what already exists. In this case, what already seems to exist is modbus, but that fundamentally has the issue of only being able to respond to commands that are sent by a consumer. There isn't any way to tell a modbus producer to just keep sending a specific piece of data until either I tell it to stop, or it runs out of said data. The other alternative to look at is, CANopen, which does allow these requirements, but at only the speed of a CANbus, which is far too slow for especially gathering a large quantity of logs, let alone not exactly being the nicest peripheral to connect to a computer.

So my suggestion: A pub/sub native protocol that allows for both rapid debugging and device control





In the process of implementing this, the major problem that I'm having is that of sizing. No doubt there will be more issues later on, but two question for me stand out:

1. How should the frame header be sized? So far I like the idea of having an exactly 64 bit header, however that may become difficult considering how much data is taken up by the Sequence ID (currently 28 bits) and the magic bytes (currently 16 bits), leaving me with only 20 bits to work with.
2. Should I include payload length in my header? I believe so, for the sake of forwards compatibility (being able to mark a message as X bytes long comes in useful if you don't actually know the command), but it seems like a bit of a waste of space if the payloads are of a known length, defined by the protocol itself.
