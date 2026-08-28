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

To specify that, for any given `Packet`, it requires that a `Read`able object is provided, while allowing `PacketData` to remain free from this constraint. This is important because I want to be able to create two types of `PacketData` structs (although I think I might rename this to `Packet` and what is currently "`Packet`" to `PacketRead` instead), one for when I'm reading from a stream, and one from when I've created it 

I have come to the conclusion that I should instead take a different approach to this. First I'll start with the actual problem: I want to parse data that contains a metadata field and optionally a "dynamic" list of elements, in that the number of elements is defined by a header in the metadata field. Since we can't use alloc, we will instead use an iterator to read the elements as they come in. This gives the following:

```rust
struct Payload<M, T, I: Iterator<Item = T>> {
    metadata: M,
    fields: I
}
```

Now, let's define the metadata field. Let's add a size to it, and specify that it must be able to be formed from a sequence of bytes of that length:

```rust
trait MetadataField<const SIZE: usize>: AsRef<[u8]> + From<[u8; SIZE]> {}

struct Payload<
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    T,
    I: Iterator<Item = T>,
> {
    metadata: M,
    fields: I,
}
```

Then we'll do something similar for the fields. I'll also add in a requirement to provide the number of fields (for us to use later):

```rust
trait FrameField<const SIZE: usize>: AsRef<[u8]> + From<[u8; SIZE]> {}
trait MetadataField<const SIZE: usize>: AsRef<[u8]> + From<[u8; SIZE]> {
    fn num_fields(&self) -> usize;
}

struct Payload<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    T: FrameField<FIELD_SIZE>,
    I: Iterator<Item = T>,
> {
    metadata: M,
    fields: I,
}
```

Then we need a reader to grab all of that data from:

```rust
struct Payload<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    T: FrameField<FIELD_SIZE>,
    I: Iterator<Item = T>,
    R: Read
> {
    metadata: M,
    fields: I,
    reader: R
}
```

Now, since the metadata must either be stored or, well, not yet stored, let's create a value to store that status, using, of course, the type-state pattern:

```rust
trait MetadataState {}
struct Cached;
impl MetadataState for Cached {}
struct UnCached;
impl MetadataState for UnCached {}

struct MetadataCache<const SIZE: usize, S: MetadataState, M: MetadataField<SIZE>> {
    _state: PhantomData<S>,
    metadata: MaybeUninit<M>
}
struct Payload<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    S: MetadataState,
    T: FrameField<FIELD_SIZE>,
    I: Iterator<Item = T>,
    R: Read,
> {
    metadata: MetadataCache<METADATA_SIZE, S, M>,
    fields: I,
    reader: R,
}
```

I'm using MaybeUninit here since I'm using `_state` as a marker, similar to `Option`, except with compile-time guarantees. Thus we establish that Metadata may be either Cached or Uncached. To make it a little easier to use, let's implement both loading the data, and the Deref trait for when it has been loaded.

```rust
struct MetadataCache<const SIZE: usize, S: MetadataState, M: MetadataField<SIZE>> {
    _state: PhantomData<S>,
    metadata: MaybeUninit<M>,
}

impl<const SIZE: usize, M: MetadataField<SIZE>> MetadataCache<SIZE, UnCached, M> {
    pub const fn new() -> Self {
        Self {
            _state: PhantomData,
            metadata: MaybeUninit::uninit(),
        }
    }
  
    pub fn load<R: Read>(self, reader: &mut R) -> Result<MetadataCache<SIZE, Cached, M>, R::Error> {
        let mut buf = [0; SIZE];
        reader.read_exact(&mut buf).unwrap();
        Ok(MetadataCache {
            _state: PhantomData,
            metadata: MaybeUninit::new(M::from(buf)),
        })
    }
}

impl<const SIZE: usize, M: MetadataField<SIZE>> Deref for MetadataCache<SIZE, Cached, M> {
    type Target = M;

    fn deref(&self) -> &Self::Target {
        unsafe { self.metadata.assume_init_ref() }
    }
}
```

The key things here are that firstly, the only way to construct Metadata with `MaybeUninit::uninit()` is in the `UnCached` state, which only implements `load`, and **not** `deref`. This guarantees that to get `Cached` `MetadataCache` (and therefore `Deref`) is to call `load`, ensuring safety (ignore the `unwrap` for now. We'll get to that later).

Now let's move on to the `fields`. Since we have the number of fields which are expected to be generated, given an iterator, we know that we will have at least one common field, which is the number of fields remaining. As such, let's remove that `I: Iterator<T>` from the `Payload`, and replace it with our own custom `Iterator`. Furthermore, we'll actually be removing the `fields`, uh, field, from the `Payload`, since it's something that we'll need to generate on the fly, rather than pre-allocate. So we use a marker to define it:

```rust
struct Payload<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    S: MetadataState,
    T: FrameField<FIELD_SIZE>,
    R: Read,
> {
    metadata: MetadataCache<METADATA_SIZE, S, M>,
    _field_iterator_marker: PhantomData<FieldIterator<FIELD_SIZE, T, R>>,
    reader: R,
}


struct FieldIterator<const SIZE: usize, T: FrameField<SIZE>, R: Read> {
    elements_remaining: usize,
    reader: R,
    _frame_type: PhantomData<T>
}
```

To actually construct this iterator though, we must first ensure that we have the correct state. Namely, the `metadata` has to be `Cached`. Care to guess what this means? More type states! (The boilerplate does get annoying though, I must admit)

```rust
impl<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    T: FrameField<FIELD_SIZE>,
    R: Read,
> Payload<FIELD_SIZE, METADATA_SIZE, M, Cached, T, R>
{
    pub fn into_iter(self) -> FieldIterator<FIELD_SIZE, T, R> {
        FieldIterator {
            elements_remaining: self.metadata.num_fields(),
            reader: self.reader,
            _frame_type: PhantomData,
        }
    }
}

```

You'll note here that I'm also intentionally removing the metadata from the `FieldIterator` when we consume the `Payload`. It's entirely possible to keep it, but personally I don't like the mess that it makes in the `FieldIterator`, and in any case I think it's fair enough to just `Copy` it if you *really* need it. For the sake of it though, here's what it would look like if we did:

```rust
impl<const SIZE: usize, M: MetadataField<SIZE>> MetadataCache<SIZE, Cached, M> {
    pub fn into_inner(self) -> M {
        unsafe { self.metadata.assume_init() }
    }
}
struct Payload<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    S: MetadataState,
    T: FrameField<FIELD_SIZE>,
    R: Read,
> {
    metadata: MetadataCache<METADATA_SIZE, S, M>,
    _field_iterator_marker: PhantomData<FieldIterator<FIELD_SIZE, METADATA_SIZE, T, R, M>>,
    reader: R,
}

impl<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: MetadataField<METADATA_SIZE>,
    T: FrameField<FIELD_SIZE>,
    R: Read,
> Payload<FIELD_SIZE, METADATA_SIZE, M, Cached, T, R>
{
    pub fn into_iter(self) -> FieldIterator<FIELD_SIZE, METADATA_SIZE, T, R, M> {
        FieldIterator {
            elements_remaining: self.metadata.num_fields(),
            metadata: self.metadata.into_inner(),
            reader: self.reader,
            _frame_type: PhantomData,
        }
    }
}

struct FieldIterator<const FIELD_SIZE: usize, const METADATA_SIZE: usize, T: FrameField<FIELD_SIZE>, R: Read, M: MetadataField<METADATA_SIZE>> {
    elements_remaining: usize,
    metadata: M,
    reader: R,
    _frame_type: PhantomData<T>,
}

```



Now the only thing left is to iterate over the fields! It's just a matter of implementing `Iterator` for the `FieldIterator`:

```rust
impl<const SIZE: usize, T: FrameField<SIZE>, R: Read> Iterator for FieldIterator<SIZE, T, R> {
    type Item = T;

    fn next(&mut self) -> Option<Self::Item> {
        if self.elements_remaining == 0 {
            return None;
        }
        let mut buf = [0; SIZE];
        self.reader.read_exact(&mut buf).ok()?;
        Some(T::from(buf))
    }
}
```
