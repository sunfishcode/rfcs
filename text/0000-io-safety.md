- Feature Name: `io_safety`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

This RFC declares that the Rust standard library provides a guarantee known as
*I/O safety*. I/O safety means that all I/O performed via handle values
([`RawFd`], [`RawHandle`], and [`RawSocket`]) uses handle values that are
explicitly returned from the OS, and occurs within the lifetime the OS
associates with them.

This RFC makes no code changes in Rust or its standard library. This just
provides documentation explaining why the functions [`FromRawFd::from_raw_fd`],
[`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`] are
`unsafe`.

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
 - A new section in the ["Keyword unsafe"] page in the
   [Rust reference documentation] documenting the new "I/O safety" concept.
 - Revised documentation comments for [`FromRawFd::from_raw_fd`],
   [`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`],
   explaining why they're `unsafe` in terms of the new rationale.
 - Similar documentation in other Rust documentation describing `unsafe`.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Rust has low-level data types, [`RawFd`] on Unix-like platforms, and
[`RawHandle`] and [`RawSocket`] on Windows, which represent resource
handles provided by the OS. These are very simple types which don't
provide any behavior on their own, and just represent identifiers
which can be passed to low-level OS APIs.

These raw resource handles can be thought of as raw pointers. They have many
of the same hazards; they can dangle and alias, and the consequences of using
an unintentionally aliased raw resource handle could include corrupted output
or silently lost input data. It could also mean that code in one crate could
accidentally corrupt or observe private data in another crate. Protection from
these hazards is called *I/O safety*.

Rust also has high-level datatypes such as [`File`] and [`TcpStream`] which
are wrappers around these low-level OS resource handles, providing high-level
interfaces on top of OS APIs. These high-level datatypes implement the traits
[`FromRawFd`] on Unix-like platforms, and [`FromRawHandle`] and
[`FromRawSocket`] on Windows, which provide functions which wrap a low-level
value to produce a high-level value.

These functions are `unsafe`, since they are unable to guarantee I/O safety.
The type system is insufficient to ensure that the handles passed in are valid.
For some examples:

```rust
    use std::fs::File;
    use std::os::unix::io::FromRawFd;

    // Create a file.
    let file = File::open("data.txt")?;

    // If it happens that `file`'s internal has the value 7, and we correctly
    // guess that here, we will have a forged file descriptor which will let
    // us do I/O as if through `file` but without needing access to `file`.
    // Consequently, `from_raw_fd` needs to be `unsafe`.
    let forged = unsafe { File::from_raw_fd(7) };

    // Obtain a copy of `file`'s inner raw handle.
    let raw_fd = file.as_raw_fd();

    // Release the original file, ending the resource's lifetime.
    drop(file);

    // Further uses of `raw_fd`, which was `file`'s inner raw handle, would be
    // outside the lifetime the OS associated with it. This could lead to
    // it aliasing other otherwise unrelated `File` instances, such as
    // `another` here. Consequently, `from_raw_fd` needs to be `unsafe`.
    let dangling = unsafe { File::from_raw_fd(raw_fd) };
    let another = File::open("another.txt")?;
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
## Unsafe I/O

In addition to memory safety, Rust's standard library also guarantees
*I/O safety*. I/O safety means that all I/O performed via handle values
([`RawFd`], [`RawHandle`], and [`RawSocket`]), including closing them,
uses handle values that are explicitly returned from the OS, and occurs
within the lifetime the OS associates with them.

For example, on Unix-like platforms, an I/O handle is a plain integer type
([`RawFd`]), so Rust's type system doesn't prevent program from creating a
handle value from an arbitrary integer value ("forging" a handle), or from
remembering the value of a handle after the resource is `close`d (a
"dangling" handle). Unix-type platforms typically define their behavior in
such cases, so this isn't about memory safety or undefined behavior at the
[language level]. Such handles can unexpectedly alias other handles in the
program, which may lead to corrupted output, lost data on input, or leaks
in encapsulation boundaries.

Some OS's document their file descriptor allocation algorithms, however use
of these algorithms to predict file descriptor values is still considered
forging, and does not produce a handle value "explicit returned from the OS".

Functions accepting arbitrary raw I/O handle values ([`RawFd`], [`RawHandle`],
or [`RawSocket`]) are marked `unsafe` if they can lead to any I/O being
performed on those handles through safe APIs.
```

Revise the parts of the documentation comments for [`FromRawFd::from_raw_fd`],
[`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`]
explaining the use of `unsafe` to say the following:

```markdown
This function is also unsafe as it may violate [I/O safety]. Callers must
ensure that the handle value passed in is a value returned from the OS and
that the resulting value won't outlive the lifetime the OS associates
with that handle value. Arbitrary handle values or values associated with
resources outside their lifetime could alias encapsulated handle values
elsewhere in a program, leading to corrupted output, lost input data, or
encapsulation leaks.
```

And, add similar text to [The Rust Book].

[language level]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
["Unsafe abilities"]: https://doc.rust-lang.org/stable/std/keyword.unsafe.html#unsafe-abilities
["Keyword unsafe"]: https://doc.rust-lang.org/stable/std/keyword.unsafe.html
[The Rust Book]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
[Rust reference documentation]: https://doc.rust-lang.org/std/keyword.unsafe.html

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
encapsulation, which will benefit more users.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Handles are plain data

The main alternative would be to say that raw resource handles are plain data,
with no concept of I/O safety and no relationship to OS resource lifetimes.
The main benefit of this approach would be to require fewer `unsafe` blocks in
use cases that work with raw resource handles. On Unix-like platforms at
least, this wouldn't ever lead to memory unsafety or undefined behavior,
because POSIX defines the behavior when I/O is attempted on invalid file
descriptors or when there are concurrent accesses to the same file descriptor.

However, most Rust code does not interact with raw resource handles directly.
This is a good thing, independently of this RFC, because resources ultimately do
have lifetimes, so most Rust code will always be better off using higher-level
types which manage these lifetimes automatically and provide better ergonomics
in many other respects.

So this alternative would at best making raw resource handles marginally more
ergonomic for uncommon use cases. This would be a small benefit, and may even
be a downside, if it ends up helping more people to write code that works with
raw resource handles when they don't need to.

## Say that handles can be *owned*

`File` acts like it owns its handle, in the sense that it frees the resources
when it's dropped. However, ownership in Rust implies *exclusive* ownership,
and that's not strictly required here.

There are even real-world use cases which effectively do things like this:

```rust
   fn foo(file: File) -> io::Result<()> {
      let raw_fd = file.as_raw_fd();
      let temp = ManuallyDrop::new(unsafe { File::from_raw_fd(raw_fd) });
      let mut buf = Vec::new();
      temp.read_to_end(&mut buf)?;
   }
```

`temp` here does not *exclusively* own its resources, but this isn't a
serious hazard in this case. The `unsafe` here covers the fact that
`temp` can't outlive `file`'s handle, since this isn't enforced by the
type system. Other than that, this program is fine.

The problems we're trying to solve here aren't about exclusivity per se,
but about avoiding accidental aliasing that can happen as a result of
forging and dangling, so exclusive ownership isn't the tool we need.

## Do nothing

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
values altogether, such as in [C#], [Java], and others. Making it `unsafe`
to perform I/O through a given raw resource handle value would let safe Rust
have the same guarantees as those effectively provided by such memory-safe
languages.

[C#]: https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0
[Java]: https://docs.oracle.com/javase/7/docs/api/java/io/File.html?is-external=true

# Unresolved questions
[unresolved-questions]: #unresolved-questions

How should we word the safety guarantees? The
[reference-level explanation](#reference-level-explanation) has some proposed
wording, but there is certainly room to discuss other ways to word these.

The reference-level explanation mentions [The Rust Book]. Is it important
to update this? Are there more references that we should update?

An issue that's out of scope is the issue that as of [rust-lang/rust#76969],
`RawFd` implements `FromRawFd` and doesn't own the file descriptor, meaning
that the existing comments about `FromRawFd` implementations "consuming
ownership" aren't accurate. This is an independent issue however, and
can be addressed in the future independently of the solution that comes out
of this RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

Some possible future ideas that would build on this solution might include:
 - As observed in [rust-lang/rust#76969], `FromRawFd` and friends don't
   require that their implementors exclusively own their resources. It may
   be useful in the future to add new `unsafe` traits which do require this.

   The [`from_filelike`] function in the `unsafe-io` crate is an example of
   what that enables&mdash;a way to convert from any type that implements the
   exclusive-owning counterpart of `IntoRawFd` into any type that implements
   the exclusive-owning counterpart of `FromRawFd` without requiring any
   unsafe in the user code.

   This technique is used in the `posish` crate to provide safe interfaces
   for POSIX-like functionality without having raw file descriptors in the
   API, such as in [this wrapper around `posix_fadvise`].

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
