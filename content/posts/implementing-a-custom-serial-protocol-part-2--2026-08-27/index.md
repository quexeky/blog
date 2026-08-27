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

A rather frustrating realisation that I have somehow only just realised is that I'm going to have to process these packets as they come in, rather than sending the entire chunk to RAM at once, since I'd need 65KiB of free RAM to do that (which is rather uncommon!). Of course, this is the way that you should be doing it. It's just annoying. That being said, it may also be a rather interesting task, as I can probably make an iterator over the entire chunk which takes in a reader and reads enough instructions to make an instruction. Yes, I'll do that.



Oh man, this is starting to sound fun again.



## Generic Madness

Both my madness in general, but also due to my love of the type state pattern ([I learned it from here](https://blog.cesc.cool/implementing-the-state-pattern-in-rust)), my conclusion has been to restructure my `packet` struct into a `PacketData` struct which owns the following `Frame` type:

```rust
pub struct PacketData<'a, R: Read, F: FrameType> {
    pub frame: Frame<'a, F, R>,
    pub flags: Flags,
    pub id: u8,
    pub sequence_id: u24,
    pub payload_length: u16,
    stored_crc: u32,
}

struct Frame<'a, F: FrameType, R: Read> {
    marker: PhantomData<F>,
    reader: &'a mut R,
}
pub enum Packet<'a, R: Read> {
    Request(PacketData<'a, R, Request>),
    Publish(PacketData<'a, R, Publish>),
    Response(PacketData<'a, R, Response>),
}
```

As such, any generic `Packet` may be read and then matched, meanwhile each different type of `Frame` can implement its own relevant logic, which makes it far more maintainable, and ensures a much greater level of type safety.

I firmly believe that the greatest thing about Rust is its type system. While memory safety is a hugely relevant factor for security, I personally find much greater satisfaction in writing code which essentially automatically resolves itself to tell me whether what I have written is indeed valid. Interestingly, the crate which uses this to its fullest extent (in my opinion) is the `embedded-hal` crate, simply due to the fact that through extremely well thought out generics, it has provided a solution to the idea of "islands of functionality".



The relevant bit with generics for me though is that I can continuously gather more and more specific data without necessarily reading all of it at once. For example, for the SUBSCRIBE REQUEST, I can use the following fields:

```rust
pub struct PacketData<F> {
    pub frame: F,
    pub flags: Flags,
    pub id: u8,
    pub sequence_id: u24,
    pub payload_length: u16,
    stored_crc: u32,
}
pub enum Packet<'a, R: Read> {
    Request(PacketData<Request<'a, R>>),
    Publish(PacketData<Publish>),
    Response(PacketData<Response>),
}


```
