---
title: Designing a custom Serial Protocol (part 3)
date: 2026-08-28T17:29:00+10:00
description: Implementing the Write side of Payloads
cover:
  relative: true
showToc: true
---
When we left off last time, the `Read` side of `Payload`s seemed like it was all good to go (I wrote a few tests to make sure that it was working properly and such). The elephant in the room still, though, is why the bounds on the `MetadataField` and `FrameField` required `AsRef<[u8]>`? We didn't use it at all!

```rust
pub trait MetadataField<const SIZE: usize>: AsRef<[u8]> + From<[u8; SIZE]> {
    fn num_fields(&self) -> usize;
}
pub trait FrameField<const SIZE: usize>: AsRef<[u8]> + From<[u8; SIZE]> {}
```



Finally, I'll be putting this to good use today - time to write `Payload`s as binary objects!



One key thing to note:
