---
title: Designing a custom Serial Protocol (part 3)
date: 2026-08-28T17:29:00+10:00
description: Implementing the Write side of Payloads
cover:
  relative: true
showToc: true
---
When we left off last time, the `Read` side of `Payload`s seemed like it was all good to go. The elephant in the room still, though, is what about creating the payloads? Right now we need to just guess at what the bytes are that we want. That seems...error-prone at best. 

So let's start writing some bytes!

I'm tempted to make a single monolithic struct with the `Payload`, but to be honest, that seems like it'd be more hassle than it's worth. Instead, I'll start by separating files into `read` and `write` sections. 

```
.
├── lib.rs
├── read
│   ├── fields.rs
│   ├── metadata.rs
│   ├── mod.rs
│   └── payload.rs
└── write
    └── mod.rs
```

Since read and write are just two sides of the same coin, most of what we did last time can actually be copied over with slight changes to ensure that we can `Write` the fields instead of `Read`, so I'll go through it very quickly.

```rust
pub trait WritableFrameField<const SIZE: usize> {
    fn write_to<W: Write>(self, writer: &mut W) -> Result<(), W::Error>;
}

pub trait WritableMetadataField<const SIZE: usize> {
    fn num_fields(&self) -> usize;
    fn write_to<W: Write>(self, writer: &mut W) -> Result<(), W::Error>;
}


```

Since this time we need to be able to turn fields into bytes, we'll require the `Writable` fields to actually write the relevant data to a buffer. Note that another option for this is `AsRef<[u8]>` , and then simply writing the `field.as_ref()` to a `writer`, but allowing the struct to implement the `write` itself gives us much more flexibility.

Now, for the interface of this code, I'm thinking I'd personally like to use it like so:

```rust
let mut payload = WritablePayload::new(writer);
payload.begin(metadata);
for packet in my_packets {
    payload.write_field(packet)
}
payload.finish();
```

Where `write_field` and `finish` check that the correct number of fields have been written. My reason for wanting to separate the `new` function from the `begin` function is the same reason why we separated `load` from `new` in `MetadataCache` - in general, I'm of the opinion that creating an object sohuld never do anything beyond loading the data required to do "things," but should never do any heavy operations - it's better to allow specificity. 

In any case, since we're offloading everything until later, this initially gives us a struct like so:

```rust
pub trait MetadataWriteState {}
struct Written {
    num_elements: usize,
}
impl MetadataWriteState for Written {}
struct NotWritten;
impl MetadataWriteState for NotWritten {}

pub struct WritablePayload<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: WritableMetadataField<METADATA_SIZE>,
    S: MetadataWriteState,
    T: WritableFrameField<FIELD_SIZE>,
    W: Write,
> {
    _metadata_type: PhantomData<M>,
    metadata_state: S,
    _frame_type: PhantomData<T>,
    writer: W,
}
```

Since we need to know the size of `metadata` and `frame` so that we can have our runtime guarantees that we're writing the correct amount of data, we're keeping the `METADATA_SIZE` and `FIELD_SIZE` bytes. The `MetadataWriteState` is, well, for telling us whether the `metadata` has been written yet, but also, if it has, then telling us exactly how many elements there are in it. Then the `new` and `begin` functions can operate on `NotWritten`:

```rust
impl<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: WritableMetadataField<METADATA_SIZE>,
    T: WritableFrameField<FIELD_SIZE>,
    W: Write,
> WritablePayload<FIELD_SIZE, METADATA_SIZE, M, NotWritten, T, W>
{
    pub fn new(writer: W) -> Self {
        Self {
            _metadata_type: PhantomData,
            metadata_state: NotWritten,
            _frame_type: PhantomData,
            writer,
        }
    }
    pub fn begin(
        mut self,
        metadata: M,
    ) -> WritablePayload<FIELD_SIZE, METADATA_SIZE, M, Written, T, W> {
        let num_fields = metadata.num_fields();
        metadata.write_to(&mut self.writer);

      WritablePayload {
            _metadata_type: PhantomData,
            metadata_state: Written { num_fields },
            _frame_type: PhantomData,
            writer: self.writer,
        }
    }
}

```

Notice that this is where the `write_to` becomes useful. 

I'll also add a slight amendment to the `begin` function here. Semi-arbitrarily, my payload will have a maximum length of 64KiB. As such, the `begin` function, once we have the field `metadata`, will be able to tell us if we're planning on writing too much data by using the `METADATA_SIZE` const. To specify this, I'll add a new error type:

```rust
use embedded_io::ErrorType;

pub enum Error<E: ErrorType> {
    TooMuchPlannedData,
    Other(E)
}

impl<E: ErrorType> From<E> for Error<E> {
    fn from(value: E) -> Self {
        Error::Other(value)
    }
}
```

Which we will then use in the `begin` function:

```rust
pub fn begin(
    mut self,
    metadata: M,
) -> Result<WritablePayload<FIELD_SIZE, METADATA_SIZE, M, Written, T, W>, Error<W::Error>>
where
    <W as ErrorType>::Error: ErrorType,
{
    let num_fields = metadata.num_fields();
    if num_fields * FIELD_SIZE >= 65536 {
        return Err(Error::TooMuchPlannedData);
    }
    metadata.write_to(&mut self.writer)?;

    Ok(WritablePayload {
        _metadata_type: PhantomData,
        metadata_state: Written { num_fields },
        _frame_type: PhantomData,
        writer: self.writer,
    })
}

```

(I'm not entirely why I needed to specify that `W` is of `ErrorType` since it's already a requirement of `Write`, but I digress - I think I have comments enabled on these things if anyone wants to let me know).

Similarly for writing fields, the only difference is that we don't want to consume `self` until we run `finish`. I'll also include two new `Error` variants: `InsufficientDataWritten` and `ExcessData`:



```rust
impl<
    const FIELD_SIZE: usize,
    const METADATA_SIZE: usize,
    M: WritableMetadataField<METADATA_SIZE>,
    T: WritableFrameField<FIELD_SIZE>,
    W: Write,
> WritablePayload<FIELD_SIZE, METADATA_SIZE, M, Written, T, W>
where
    <W as ErrorType>::Error: ErrorType,
{
    pub fn write_field(&mut self, field: T) -> Result<(), Error<W::Error>> {
        if self.metadata_state.fields_to_write == 0 {
            return Err(Error::ExcessData);
        }
        self.metadata_state.fields_to_write -= 1;
        field.write_to(&mut self.writer)?;
        Ok(())
    }
    pub fn finish(mut self) -> Result<(), Error<W::Error>> {
        if self.metadata_state.fields_to_write != 0 {
            return Err(Error::InsufficientDataWritten);
        }
        self.writer.flush()?;
        Ok(())
    }
}

```
