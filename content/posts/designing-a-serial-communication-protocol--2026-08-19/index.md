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

So far my conclusion to this gives a packet defined as such:

1. Magic bytes: 16 bits for a pair of magic synchronisation bytes. Any pair of bytes like these will be assumed to mark the beginning of a frame, which is useful for synchronisation, although it does mean that any similar bytes will have to be escaped (this is why I'm using 16 bits instead of just 8). Note that these are considered to be a prefix rather than necessarily part of the header itself, and so isn't counted towards the length of the parsed frame header  
2. Frame Header: A fixed length segment of a packet that defines what will be read. This contains:

   * Major Version (4 bits): The major version of the sender. Useful for allowing custom definitions.
   * Frame type (4 bits): To allow various types of frames, such as REQUEST (a consumer requesting data from a server), RESPONSE (a server providing a response), and PUBLISH (a packet in a continuous stream of data that does not require REQUESTs)
   * Flag (4 bits): Marks some characteristic about a Frame
   * ID (8 bits): One of two ID fields. Corresponds to Command ID in REQUEST frames, or Subscribe ID in PUBLISH frames
   * Sequence ID (28 bits): Second identity field. Corresponds to Request ID in REQUEST frames, or absolute position in PUBLISH frames
   * Payload Length (16 bits): Length of the data payload (where payload is payload_len bytes)
3. Payload: The contents of the packet. Exactly payload_len bytes long
4. CRC-32 (Or maybe 16? I think 32 is better though, since either the payload length is very long for PUBLISH messages, or short but we need to be absolutely sure of no errors like in REQUESTs)



# Implementation

Of course, everything is better in Rust. As such, this MVP will also be written as a Rust library initially, before I then use it in an MC. I don't think that it's actually a terribly difficult process. I've recently found the [bilge](https://docs.rs/bilge/latest/bilge/) crate, which seems perfectly suited to this purpose. The Frame Header looks a little something like this:

```rust
#[bitsize(64)]
#[derive(FromBits, Clone, Copy)]
pub struct Header {
    major_version: u4,
    frame_type: FrameType,
    flag: Flag,
    id: u8,
    sequence_id: u24,
    payload_length: u16,
}

#[bitsize(4)]
#[derive(FromBits, Clone, Copy)]
pub enum FrameType {
    Request = 0x1,
    Response = 0x2,
    Publish = 0x3,
    #[fallback]
    Reserved,
}

#[bitsize(8)]
#[derive(FromBits, Clone, Copy)]
pub struct Flag {
    requires_ack: bool,
    finish: bool,
    reserved: bool,
    reserved: u5,
}
```

(I really love Rust's type checking. In contrast, the implicit type casting in C is atrocious, and I cannot fathom why it is reasonable to cast a 32 bit pointer to a uint16_t, which I accidentally did for a while in a work project :/ )



## REQUEST Forms

As much as I love JSON, it's simply too inefficient at what it does. As such, after very little digging, I've found the ["Concise Binary Object Representation" (CBOR)](https://cbor.io/), which I'll be using for any data that needs to be self describing. For example:

### METADATA

The METADATA type for a REQUEST must provide a basic set of specifications about a device, which at minimum means:

* Vendor
* Manufacturer
* Serial Number

These are the fields that I personally consider the be an absolute minimum. Since this protocol is extensible, these fields will be stored in CBOR, and manufacturers may include other (unreserved) fields in this response.
