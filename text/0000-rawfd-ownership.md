- Feature Name: `rawfd_ownership`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC declares that [`RawFd`], [`RawHandle`], and [`RawSocket`] represent
resources that can be owned and exclusively controlled as part of a type's
safety invariant. This will provide justification and expanded documentation
for why [`FromRawFd::from_raw_fd`], [`FromRawHandle::from_raw_handle`], and
[`FromRawSocket::from_raw_socket`] are `unsafe`, and also provide guidance
for any future standard library or third-party crate APIs that operate on
`RawFd` `RawHandle`, or `RawSocket` values.

This RFC makes no code changes. [`FromRawFd::from_raw_fd`],
[`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`] are
all already `unsafe`, and there are no other functions in the Rust standard
library which would be affected.

[`RawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/type.RawFd.html
[`RawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawHandle.html
[`RawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawSocket.html
[`FromRawFd::from_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html#tymethod.from_raw_fd
[`FromRawHandle::from_raw_handle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html#tymethod.from_raw_handle
[`FromRawSocket::from_raw_socket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html#tymethod.from_raw_socket

# Motivation
[motivation]: #motivation

The purpose is to answer a question that has come up a [few] [times] about why
`from_raw_fd` is `unsafe`, to provide clear documentation about it and provide
useful guarantees to Rust programs, and to give clear guidance to the Rust
community about the relationship between `unsafe` and I/O.

[few]: https://github.com/rust-lang/rust/issues/72175
[times]: https://users.rust-lang.org/t/why-is-fromrawfd-unsafe/39670

This supports use cases that work with `RawFd`, `RawHandle`, and `RawSocket`
types directly, such as the [`nix`], [`unsafe-io`], and [`posish`] crates,
to give them clear guidance on when functions working with these types
should be marked `unsafe`.

[`nix`]: https://crates.io/crates/nix
[`unsafe-io`]: https://crates.io/crates/unsafe-io
[`posish`]: https://crates.io/crates/posish

The expected outcomes are:
 - A new entry in the ["Behavior considered undefined"] section of
   The Rust Reference stating that functions which assume ownership or
   rely in the validity of [`RawFd`], [`RawHandle`], and [`RawSocket`]
   values are `unsafe`.
 - Revised documentation comments for [`FromRawFd::from_raw_fd`],
   [`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`],
   explaining why they're `unsafe` in terms of the new rationale.
 - Add documentation to Rust language documentation describing `unsafe`
   mentioning this new guarantee.

["Behavior considered undefined"]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Rust has low-level data types, [`RawFd`] on Unix-like platforms, and
[`RawHandle`] and [`RawSocket`] on Windows, which represent resource
handles provided by the OS. These are very simple types which don't
provide any behavior on their own, and just represent identifiers
which can be passed to low-level OS APIs.

Rust also has high-level datatypes such as [`File`] and [`TcpStream`] which
are wrappers around these low-level OS resource handles, providing high-level
interfaces on top of OS APIs. They conceptually *own* their resources,
including freeing the resources when they are dropped.

These high-level datatypes implement the traits [`FromRawFd`] on Unix-like
platforms, and [`FromRawHandle`] and [`FromRawSocket`] on Windows, which
provide functions which wrap a low-level value to produce a high-level value.

```rust
    use std::fs::File;
    use std::mem::forget;

    // Create a file.
    let file = File::open("data.txt");

    // Obtain a copy of its inner raw handle.
    let raw_fd = file.as_raw_fd();

    // Construct a new file using `from_raw_fd`, which is `unsafe`. After
    // this, both `file` and `another` think they own the raw resource!
    let another = unsafe { File::from_raw_fd(raw_fd) };

    // So don't drop `another`, because that would make `file`'s handle
    // dangle!
    forget(another);
```

Functions like `from_raw_fd` are considered `unsafe` because raw resource
handles can be thought of as raw pointers. They have many of the same hazards;
they can dangle and alias, and the consequences of using a dangling or
unintentionally aliased raw resource handle could include corrupted I/O or
lost data. It could also mean that code in one crate could accidentally corrupt
or observe private data in another crate.

[`File`]: https://doc.rust-lang.org/stable/std/fs/struct.File.html
[`TcpStream`]: https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html
[`FromRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html
[`FromRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html
[`FromRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add the following wording to the ["Behavior considered undefined"] section of
The Rust Reference:

> * Performing I/O (such as `read` or `write`) a dangling or unallocated raw
    I/O resource handle.

Revise the parts of the documentation comments for [`FromRawFd::from_raw_fd`],
[`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`]
explaining the use of `unsafe` to say the following:

> This function is also unsafe as the primitives currently returned have the
> contract that they are the sole owner of the file descriptor they are
> wrapping. Usage of this function could accidentally allow violating this
> contract which can cause I/O corruption or broken encapsulation in code that
> relies on it being true.

Say that "a resource that can be owned and exclusively controlled as part of
a type's safety invariant" in various places where the meaning of `unsafe`
is defined, possibly including [The Rust Book], the
[Rust reference documentation], and [The Rustonomicon].

[The Rust Book]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
[Rust reference documentation]: https://doc.rust-lang.org/std/keyword.unsafe.html
[The Rustonomicon]: https://doc.rust-lang.org/nomicon/what-unsafe-does.html

# Drawbacks
[drawbacks]: #drawbacks

The main drawbacks are:
 - This will imply that crates with APIs that use file descriptors a lot,
   notably `nix`, should change many functions to be `unsafe`, and code using
   those APIs would need to be updated.
 - This would rule out adding safe versions of `FromRawFd`, `FromRawHandle`,
   and `FromRawSocket`, which would be convenient for some use cases.

While `nix` is a popular crate, it is a very low-level crate and this RFC
argues that the more important goal here is to strengthen Rust's safety and
encapsulation, which will benefits more users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main alternative would be to say that raw resource handles are plain data,
and no library may attach any special invariants to them or consider them
exclusively owned. The main benefit of this approach would be to require
fewer `unsafe` blocks in use cases that work with raw resource handles.
On Unix-like platforms at least, this wouldn't ever lead to memory unsafety
or undefined behavior, because POSIX defines the behavior when I/O is
attempted on invalid file descriptors or when there are concurrent accesses
to the same file descriptor.

However, most Rust code does not interact with raw resource handles directly.
This is a good thing, independently of this RFC, because resources ultimately do
have lifetimes, so most Rust code will always be better off using higher-level
types which manage these lifetimes automatically and provide better ergonomics
in many other respects.

So this alternative would at best making raw resource handles marginally more
ergonomic for uncommon use cases. This would be a small benefit, and may even
be a downside, if it ends up helping more people to write code that works with
raw resource handles when they don't need to.

The other alternative would be to do nothing. [`FromRawFd::from_raw_fd`] and
friends have confusing comments, the ecosystem is inconsistent in how it uses
these features, and it's ambiguous whether the Rust language makes any
guarantees about whether I/O can be fully encapsulated, but for most users
these aren't urgent practical problems.

The benefits to the RFC are more about refining one aspect of Rust's overall
safety guarantee, and it's difficult to quantify the practical advantages.

# Prior art
[prior-art]: #prior-art

Most memory-safe programming languages have safe abstractions around raw
resource handles. Most often, they simply avoid exposing the raw resource
values altogether, such as in [C#], [Java], and others. As far as I'm
aware, Rust is unique in being both memory-safe and exposing raw resource
handles. Making it `unsafe` to convert a raw resource handle into an
owned one would let safe Rust have the same guarantees as those effectively
provided by other memory-safe languages.

[C#]: https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0
[Java]: https://docs.oracle.com/javase/7/docs/api/java/io/File.html?is-external=true

# Unresolved questions
[unresolved-questions]: #unresolved-questions

How should we word the safety guarantees? The
[reference-level explanation](#reference-level-explanation) has some proposed
wording, but there is certainly room to discuss other ways to word these.

The reference-level explanation mentions [The Rust Book], the
[Rust reference documentation], and [The Rustonomicon]; is it important
to update all of these? Are there more references that we should update?

An issue that's out of scope is the issue that as of [rust-lang/rust#76969],
`RawFd` implements `FromRawFd` and doesn't own the file descriptor, meaning
that the existing comments about `FromRawFd` implementations being
"the sole owner" aren't accurate. This is an independent issue however, and
can be addressed in the future independently of the solution that comes out
of this RFC.

[rust-lang/rust#76969]: https://github.com/rust-lang/rust/pull/76969

# Future possibilities
[future-possibilities]: #future-possibilities

Some possible future ideas that would build on this solution might include:
 - Variants of `FromRawFd` and related traits which would be `unsafe` traits
   and which *would* imply ownership.

   The [`from_filelike`] function in the `unsafe-io` crate is an example of
   what that enables&mdash;a way to convert from any type that implements the
   owning counterpart of `IntoRawFd` into any type that implements the owning
   counterpart of `FromRawFd` without requiring any unsafe in the user code.

   This technique is used in the `posish` crate to provide safe interfaces
   for POSIX-like functionality without having raw file descriptors in the
   API, such as in [this wrapper around `posix_fadvise`].

 - A fine-grained capability-oriented security model for Rust, built on the
   fact that, with this new guarantee, the high-level wrappers around raw
   resource handles are unforgeable in safe Rust.

[`from_filelike`]: https://docs.rs/unsafe-io/0.6.2/unsafe_io/trait.FromUnsafeFile.html#method.from_filelike
[this wrapper around `posix_fadvise`]: https://docs.rs/posish/0.6.1/posish/fs/fn.fadvise.html

# Thanks
[thanks]: #thanks

Thanks to Ralf Jung ([@RalfJung]) whose [comment] led to my current understanding
of this topic, who more recently [suggested] the idea of an RFC for this and
contributed some of the wording used above.

[@RalfJung]: https://github.com/RalfJung
[comment]: https://github.com/bytecodealliance/wasi/issues/8#issuecomment-525711065
[suggested]: https://github.com/rust-lang/rust/issues/72175#issuecomment-815596649
