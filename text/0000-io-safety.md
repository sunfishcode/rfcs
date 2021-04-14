- Feature Name: `io_safety`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Improve the explanation of why the functions
[`FromRawFd::from_raw_fd`], [`FromRawHandle::from_raw_handle`], and
[`FromRawSocket::from_raw_socket`], are `unsafe`.

In support of this, declare that the Rust standard library guarantees
*I/O safety*. This means all I/O performed via raw handle values ([`RawFd`],
[`RawHandle`], and [`RawSocket`]) uses values that are explicitly returned
from the OS, and occurs within the lifetimes the OS associates with them.

The affected functions are already `unsafe`, so there are no changes to Rust's
own code here. This RFC just proposes a new explanation for *why* they're
unsafe, and new guidance to authors of crates that operate on raw handle
values.

[`RawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/type.RawFd.html
[`RawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawHandle.html
[`RawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawSocket.html
[`FromRawFd::from_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html#tymethod.from_raw_fd
[`FromRawHandle::from_raw_handle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html#tymethod.from_raw_handle
[`FromRawSocket::from_raw_socket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html#tymethod.from_raw_socket

# Motivation
[motivation]: #motivation

This proposal seeks to answer a question that has come up a [few] [times] about
why `from_raw_fd` is `unsafe` in a way that provides useful guarantees to Rust
programs, to document it, and to give guidance to crate authors about the
relationship between `unsafe` and I/O.

This supports use cases that work with `RawFd`, `RawHandle`, and `RawSocket`
types directly, such as [`nix`], [`socket2`], [`unsafe-io`], and [`posish`]
crates, and others.

The expected outcomes are:

 - A new section in the ["Keyword unsafe"] page in the
   [Rust reference documentation] documenting the new "I/O safety" concept.
 - Revised documentation comments for [`FromRawFd::from_raw_fd`],
   [`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`],
   explaining why they're `unsafe` in terms of I/O safety.
 - Similar documentation in other Rust documentation describing `unsafe`.

[few]: https://github.com/rust-lang/rust/issues/72175
[times]: https://users.rust-lang.org/t/why-is-fromrawfd-unsafe/39670
[`nix`]: https://crates.io/crates/nix
[`socket2`]: https://crates.io/crates/socket2
[`unsafe-io`]: https://crates.io/crates/unsafe-io
[`posish`]: https://crates.io/crates/posish

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Rust has low-level types, [`RawFd`] on Unix-like platforms, and
[`RawHandle`]/[`RawSocket`] on Windows, which represent resource handles
provided by the OS. These don't provide any behavior on their own, and just
represent identifiers which can be passed to low-level OS APIs.

These raw resource handles can be thought of as raw pointers. They have many of
the same hazards; they can dangle and alias, and the consequences of using an
unintentionally aliased raw resource handle could include corrupted output or
silently lost input data. It could also mean that code in one crate could
accidentally corrupt or observe private data in another crate. Protection from
these hazards is called *I/O safety*.

Rust also has high-level types such as [`File`] and [`TcpStream`] which are
wrappers around these low-level OS resource handles, providing high-level
interfaces to OS APIs.

These high-level types also implement the traits [`FromRawFd`] on Unix-like
platforms, and [`FromRawHandle`]/[`FromRawSocket`] on Windows, which provide
functions which wrap a low-level value to produce a high-level value. These
functions are `unsafe`, since they're unable to guarantee I/O safety. The type
system is insufficient to ensure that the handles passed in are valid. For
example:

```rust
    use std::fs::File;
    use std::os::unix::io::FromRawFd;

    // Create a file.
    let file = File::open("data.txt")?;

    // Construct a `File` from an arbitrary integer value. This type checks,
    // however 7 may not identify a live resource at runtime, or it may
    // accidentally alias encapsulated resources elsewhere in the program, so
    // this requires an `unsafe` block to recognize that it's the caller's
    // responsibility to avoid these hazards.
    let forged = unsafe { File::from_raw_fd(7) };

    // Obtain a copy of `file`'s inner raw handle.
    let raw_fd = file.as_raw_fd();

    // Release `file`, ending the resource's lifetime.
    drop(file);

    // Open some unrelated file.
    let another = File::open("another.txt")?;

    // Further uses of `raw_fd`, which was `file`'s inner raw handle, would be
    // outside the lifetime the OS associated with it. This could lead to it
    // accidentally aliasing other otherwise encapsulated `File` instances,
    // such as `another`. Consequently, this requires an `unsafe` block to
    // recognize that it's the caller's responsibility to avoid these hazards.
    let dangling = unsafe { File::from_raw_fd(raw_fd) };
```

Callers must ensure that the handle value passed into `from_raw_fd` is a value
explicitly returned from the OS, and that the return value of `from_raw_fd`
won't outlive the lifetime the OS associates with the handle value.

[`File`]: https://doc.rust-lang.org/stable/std/fs/struct.File.html
[`TcpStream`]: https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html
[`FromRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html
[`FromRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html
[`FromRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Add the following subsection to the ["Unsafe abilities"] section of the
["Keyword unsafe"] page in the [Rust reference documentation]:

```markdown
## I/O Safety

In addition to memory safety, Rust's standard library also guarantees
*I/O safety*. I/O safety means that all I/O performed via raw handle values
([`RawFd`], [`RawHandle`], and [`RawSocket`]), including closing them, uses
values that are explicitly returned from the OS, and occurs within the lifetime
the OS associates with them.

For example, on Unix-like platforms, [`RawFd`] is a plain integer type, so
Rust's type system doesn't prevent program from creating a handle value from an
arbitrary integer value ("forging" a handle), or from remembering the value of
a handle after the resource is `close`d (a "dangling" handle). Unix-type
platforms typically define their behavior in such cases, so this isn't about
memory safety or undefined behavior at the [language level]. However, such
handles can unexpectedly alias other handles in the program, which may lead to
corrupted output, lost data on input, or leaks in encapsulation boundaries.

Some OS's document their file descriptor allocation algorithms, however use of
these algorithms to predict file descriptor values is still considered forging,
and doesn't produce a handle value "explicitly returned from the OS".

Functions accepting arbitrary raw I/O handle values ([`RawFd`], [`RawHandle`],
or [`RawSocket`]) are considered `unsafe` if they can lead to any I/O being
performed on those handles through safe APIs.
```

And, revise the parts of the documentation comments for
[`FromRawFd::from_raw_fd`], [`FromRawHandle::from_raw_handle`], and
[`FromRawSocket::from_raw_socket`] explaining the use of `unsafe` to say the
following:

```rust
/// This function is also `unsafe` as it isn't able to guarantee [I/O safety].
/// Callers must ensure that the handle value passed in is a value returned
/// from the OS and that the resulting value won't outlive the lifetime the OS
/// associates with that handle value. Arbitrary handle values or values
/// associated with resources outside their lifetime could alias encapsulated
/// handle values elsewhere in a program, leading to corrupted output, lost
/// input data, or encapsulation leaks.
```

And, add similar text to [The Rust Book].

[language level]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
["Unsafe abilities"]: https://doc.rust-lang.org/stable/std/keyword.unsafe.html#unsafe-abilities
["Keyword unsafe"]: https://doc.rust-lang.org/stable/std/keyword.unsafe.html
[The Rust Book]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
[Rust reference documentation]: https://doc.rust-lang.org/std/keyword.unsafe.html

# Drawbacks
[drawbacks]: #drawbacks

Drawbacks include:

 - This implies that crates with APIs that use file descriptors, such as `nix`,
   should either change such functions to be `unsafe`, or change them to use
   types that make stronger guarantees.
 - This would require changes to crates using `AsRawFd` or `IntoRawFd` to accept
   "any file-like type" or "any socket-like type", such as `socket2`'s
   [`SockRef::from`]. One option is to make those functions `unsafe`. Another
   is for those functions to use the utilities in the [`unsafe-io`] crate which
   factor out the `unsafe` parts of this pattern.
 - This would rule out adding safe versions of `FromRawFd`, `FromRawHandle`,
   and `FromRawSocket`, which would be convenient for some use cases.

While `nix` is a popular crate, it's a low-level crate and this RFC argues that
the more important goal here is to strengthen Rust's safety and encapsulation,
which will benefit more users.

 - This explicitly extends the scope of `unsafe` to something outside of just
   memory safety and undefined behavior, which Rust has long limited it to.

The scope of `unsafe` is a choice that Rust makes, rather than being derived
from any fundamental constraints. Here, the observation is that since invalid
resource handles can break encapsulation boundaries and cause spooky action at
a distance, I/O safety is sufficiently similar in spirit to memory safety that
it's worth covering.

[`SockRef::from`]: https://docs.rs/socket2/0.4.0/socket2/struct.SockRef.html#method.from

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Handles are plain data

The main alternative would be to say that raw resource handles are plain data,
with no concept of I/O safety and no relationship to OS resource lifetimes.
The main benefit of this approach would be to require fewer `unsafe` blocks in
use cases that work with raw resource handles. On Unix-like platforms at least,
this wouldn't ever lead to memory unsafety or undefined behavior.

However, most Rust code doesn't interact with raw resource handles directly.
This is a good thing, independently of this RFC, because resources ultimately do
have lifetimes, so most Rust code will always be better off using higher-level
types which manage these lifetimes automatically and provide better ergonomics
in many other respects.

So this alternative would at best making raw resource handles marginally more
ergonomic for uncommon use cases. This would be a small benefit, and may even
be a downside, if it ends up encouraging people to write code that works with
raw resource handles when they don't need to.

## Do nothing

On a practical level, the problems addressed here aren't urgent for most Rust
users today, so Rust could arguably get by without doing anything here.

However, this would leave [`FromRawFd::from_raw_fd`] with confusing comments,
inconsistency in parts of the ecosystem that use these features, and ambiguity
over whether Rust intends that I/O can be fully encapsulated.

# Prior art
[prior-art]: #prior-art

Most memory-safe programming languages have safe abstractions around raw
resource handles. Most often, they simply avoid exposing the raw resource
values altogether, such as in [C#], [Java], and others. Making it `unsafe` to
perform I/O through a given raw resource handle value would let safe Rust have
the same guarantees as those effectively provided by such memory-safe
languages.

[C#]: https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0
[Java]: https://docs.oracle.com/javase/7/docs/api/java/io/File.html?is-external=true

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Is the wording in the [reference-level explanation] the best way to describe
the safety guarantees?

The reference-level explanation mentions [The Rust Book]. Is it important to
update this? Are there other references that we should update?

An issue that's out of scope is the issue that as of [rust-lang/rust#76969],
`RawFd` implements `FromRawFd` and doesn't own the file descriptor, meaning
that the existing comments about `FromRawFd` implementations "consuming
ownership" aren't accurate. The meaning of ownership of a raw resource handle
is something that can be addressed in the future independently of the solution
that comes out of this RFC.

The rules for raw handle values in this proposal are vague about the identity
such values. What does it mean for a resource lifetime to be associated with a
handle if the handle is just an integer type? Do all integer types with the
same value share that association? The Rust [reference] defines undefined
behavior for memory in terms of [LLVM's pointer aliasing rules]; we could
conceivably need a similar concept of handle aliasing rules. This doesn't seem
necessary for present practical needs, but it could be added later if there's a
need for greater precision.

[reference]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[LLVM's pointer aliasing rules]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules

# Future possibilities
[future-possibilities]: #future-possibilities

Some possible future ideas that could build on this solution might include:

 - As observed in [rust-lang/rust#76969], `FromRawFd` and friends don't
   require that their implementors exclusively own their resources. It may be
   useful in the future to add new `unsafe` traits which do require this.

   The [`from_filelike`] function in the `unsafe-io` crate is an example of
   what that enables&mdash;a way to convert from any type that implements the
   exclusive-owning counterpart of `IntoRawFd` into any type that implements
   the exclusive-owning counterpart of `FromRawFd` without requiring any
   `unsafe` in the user code.

   This is used in the `posish` crate to provide safe interfaces for POSIX-like
   functionality without having raw file descriptors in the API, such as in
   [this wrapper around `posix_fadvise`].

 - A formal model of ownership for raw resource handles.

 - A fine-grained capability-oriented security model for Rust, built on the
   fact that, with this new guarantee, the high-level wrappers around raw
   resource handles are unforgeable in safe Rust.

[rust-lang/rust#76969]: https://github.com/rust-lang/rust/pull/76969
[`from_filelike`]: https://docs.rs/unsafe-io/0.6.2/unsafe_io/trait.FromUnsafeFile.html#method.from_filelike
[this wrapper around `posix_fadvise`]: https://docs.rs/posish/0.6.1/posish/fs/fn.fadvise.html

# Thanks
[thanks]: #thanks

Thanks to Ralf Jung ([@RalfJung]) for leading me to my current understanding
of this topic, and for encouraging and reviewing this RFC!

[@RalfJung]: https://github.com/RalfJung
