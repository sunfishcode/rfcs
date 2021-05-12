- Feature Name: `io_safety`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Close a hole in encapsulation boundaries in Rust by providing users of
`AsRawFd` and related traits guarantees about their raw resource handles, by
introducing a concept of *I/O safety* and a new `IoSafe` trait. Build on, and
provide an explanation for, the `from_raw_fd` function being unsafe.

# Motivation
[motivation]: #motivation

Rust's standard library almost provides *I/O safety*, a guarantee that if one
part of a program holds a raw handle privately, other parts cannot access it.
[`FromRawFd::from_raw_fd`] is unsafe, which prevents users from doing things
like `File::from_raw_fd(7)`, in safe Rust, and doing I/O on a file descriptor
which might be held privately elsewhere in the program.

However, there's a loophole. Many library APIs use [`AsRawFd`]/[`IntoRawFd`] to
accept values to do I/O operations with:

```rust
pub fn do_some_io<FD: AsRawFd>(input: &FD) -> io::Result<()> {
    some_syscall(input.as_raw_fd())
}
```

`AsRawFd` doesn't restrict `as_raw_fd`'s return value, so `do_some_io` can end
up doing I/O on arbitrary `RawFd` values. One can even write `do_some_io(&7)`,
since [`RawFd`] itself implements `AsRawFd`.

This can cause programs to [access the wrong resources], or even break
encapsulation boundaries by creating aliases to raw handles held privately
elsewhere, causing [spooky action at a distance].

And in specialized circumstances, violating I/O safety could even lead to
violating memory safety. For example, in theory it should be possible to make
a safe wrapper around an `mmap` of a file descriptor created by Linux's
[`memfd_create`] system call and pass `&[u8]`s to safe Rust, since it's an
anonymous open file which other processes wouldn't be able to access. However,
without I/O safety, and without permenantly sealing the file, other code in
the program could accidentally call `write` or `ftruncate` on the file
descriptor, breaking the memory-safety invariants of `&[u8]`.

This RFC introduces a path to gradually closing this loophole by introducing:

 - A new concept, I/O safety, to be documented in the standard library
   documentation.
 - A new trait, `std::io::IoSafe`.
 - New documentation for
   [`from_raw_fd`]/[`from_raw_handle`]/[`from_raw_socket`] explaining why
   they're unsafe in terms of I/O safety, addressing a question that has
   come up a [few] [times].

[few]: https://github.com/rust-lang/rust/issues/72175
[times]: https://users.rust-lang.org/t/why-is-fromrawfd-unsafe/39670
[access the wrong resources]: https://cwe.mitre.org/data/definitions/910.html
[spooky action at a distance]: https://en.wikipedia.org/wiki/Action_at_a_distance_(computer_programming)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The I/O safety concept

Rust's standard library has low-level types, [`RawFd`] on Unix-like platforms,
and [`RawHandle`]/[`RawSocket`] on Windows, which represent raw OS resource
handles. These don't provide any behavior on their own, and just represent
identifiers which can be passed to low-level OS APIs.

These raw handles can be thought of as raw pointers, with similar hazards.
While it's safe to *obtain* a raw pointer, *dereferencing* a raw pointer could
invoke undefined behavior if it isn't a valid pointer or if it outlives the
lifetime of the memory it points to. Similarly, it's safe to *obtain* a raw
handle, via [`AsRawFd::as_raw_fd`] and similar, but using it to do I/O could
lead to corrupted output, lost or leaked input data, or violated encapsulation
boundaries, if it isn't a valid handle or it's used after the `close` of its
resource. And in both cases, the effects can be non-local, affecting otherwise
unrelated parts of a program. Protection from raw pointer hazards is called
memory safety, so protection from raw handle hazards is called *I/O safety*.

Rust's standard library also has high-level types such as [`File`] and
[`TcpStream`] which are wrappers around these raw handles, providing high-level
interfaces to OS APIs.

These high-level types also implement the traits [`FromRawFd`] on Unix-like
platforms, and [`FromRawHandle`]/[`FromRawSocket`] on Windows, which provide
functions which wrap a low-level value to produce a high-level value. These
functions are unsafe, since they're unable to guarantee I/O safety. The type
system doesn't constrain the handles passed in:

```rust
    use std::fs::File;
    use std::os::unix::io::FromRawFd;

    // Create a file.
    let file = File::open("data.txt")?;

    // Construct a `File` from an arbitrary integer value. This type checks,
    // however 7 may not identify a live resource at runtime, or it may
    // accidentally alias encapsulated raw handles elsewhere in the program. An
    // `unsafe` block acknowledges that it's the caller's responsibility to
    // avoid these hazards.
    let forged = unsafe { File::from_raw_fd(7) };

    // Obtain a copy of `file`'s inner raw handle.
    let raw_fd = file.as_raw_fd();

    // Close `file`.
    drop(file);

    // Open some unrelated file.
    let another = File::open("another.txt")?;

    // Further uses of `raw_fd`, which was `file`'s inner raw handle, would be
    // outside the lifetime the OS associated with it. This could lead to it
    // accidentally aliasing other otherwise encapsulated `File` instances,
    // such as `another`. Consequently, an `unsafe` block acknowledges that
    // it's the caller's responsibility to avoid these hazards.
    let dangling = unsafe { File::from_raw_fd(raw_fd) };
```

Callers must ensure that the value passed into `from_raw_fd` is explicitly
returned from the OS, and that `from_raw_fd`'s return value won't outlive the
lifetime the OS associates with the handle.

I/O safety is new as an explicit concept, but it reflects common practices.
Rust's `std` will require no changes to stable interfaces, beyond the
introduction of a new trait and new impls for it. Initially, not all of the
Rust ecosystem will support I/O safety though; adoption will be gradual.

## The `IoSafe` trait

These high-level types also implement the traits [`AsRawFd`]/[`IntoRawFd`] on
Unix-like platforms and
[`AsRawHandle`]/[`AsRawSocket`]/[`IntoRawHandle`]/[`IntoRawSocket`] on Windows,
providing ways to obtain the low-level value contained in a high-level value.
APIs use these to accept any type containing a raw handle, such as in the
`do_some_io` example in the [motivation].

`AsRaw*` and `IntoRaw*` don't make any guarantees, so to add I/O safety, types
will implement a new trait, `IoSafe`:

```rust
pub unsafe trait IoSafe {}
```

There are no required functions, so implementing it just takes one line, plus
comments:

```rust
/// # Safety
///
/// `MyType` wraps a `std::fs::File` which handles the low-level details, and
/// doesn't have a way to reassign or independently drop it.
unsafe impl IoSafe for MyType {}
```

It requires `unsafe`, to require the code to explicitly commit to upholding I/O
safety. With `IoSafe`, the `do_some_io` example should simply add a
`+ IoSafe` to provide I/O safety:

```rust
pub fn do_some_io<FD: AsRawFd + IoSafe>(input: &FD) -> io::Result<()> {
    some_syscall(input.as_raw_fd())
}
```

## Gradual adoption

I/O safety and `IoSafe` wouldn't need to be adopted immediately, adoption
could be gradual:

 - First, `std` adds `IoSafe` with impls for all the relevant `std` types.
   This is a backwards-compatible change.

 - After that, crates could implement `IoSafe` for their own types. These
   changes would be small and semver-compatible, without special coordination.

 - Once the standard library and enough popular crates utilize `IoSafe`,
   crates could start to add `+ IoSafe` bounds (or adding `unsafe`), at their
   own pace. These would be semver-incompatible changes, though most users of
   APIs adding `+ IoSafe` wouldn't need any changes.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The I/O safety concept

In addition to the Rust language's memory safety, Rust's standard library also
guarantees I/O safety. An I/O operation is *valid* if the raw handles
([`RawFd`], [`RawHandle`], and [`RawSocket`]) it operates on are values
explicitly returned from the OS, and the operation occurs within the lifetime
the OS associates with them. Rust code has *I/O safety* if it's not possible
for that code to cause invalid I/O operations.

While some OS's document their file descriptor allocation algorithms, a handle
value predicted with knowledge of these algorithms isn't considered "explicitly
returned from the OS".

Functions accepting arbitrary raw I/O handle values ([`RawFd`], [`RawHandle`],
or [`RawSocket`]) should be `unsafe` if they can lead to any I/O being
performed on those handles through safe APIs.

Functions accepting types implementing
[`AsRawFd`]/[`IntoRawFd`]/[`AsRawHandle`]/[`AsRawSocket`]/[`IntoRawHandle`]/[`IntoRawSocket`]
should add a `+ IoSafe` bound if they do I/O with the returned raw handle.

## The `IoSafe` trait

Types implementing `IoSafe` guarantee that they uphold I/O safety. They must
not make it possible to write a safe function which can perform invalid I/O
operations, and:

 - A type implementing `AsRaw* + IoSafe` means its `as_raw_*` function returns
   a handle which is valid to use for the duration of the `&self` reference.
   If such types have methods to close or reassign the handle without
   dropping the whole object, they must document the conditions under which
   existing raw handle values remain valid to use.

 - A type implementing `IntoRaw* + IoSafe` means its `into_raw_*` function
   returns a handle which is valid to use at the point of the return from
   the call.

All standard library types implementing `AsRawFd` implement `IoSafe`, except
`RawFd`.

# Drawbacks
[drawbacks]: #drawbacks

Crates with APIs that use file descriptors, such as [`nix`] and [`mio`], would
need to migrate to types implementing `AsRawFd + IoSafe`, use crates providing
equivalent mechanisms such as [`unsafe-io`], or change such functions to be
unsafe.

Crates using `AsRawFd` or `IntoRawFd` to accept "any file-like type" or "any
socket-like type", such as [`socket2`]'s [`SockRef::from`], would need to
either add a `+ IoSafe` bound or make these functions unsafe.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Handles as plain data

The main alternative would be to say that raw handles are plain data, with no
concept of I/O safety and no inherent relationship to OS resource lifetimes. On
Unix-like platforms at least, this wouldn't ever lead to memory unsafety or
undefined behavior.

However, most Rust code doesn't interact with raw handles directly. This is a
good thing, independently of this RFC, because resources ultimately do have
lifetimes, so most Rust code will always be better off using higher-level types
which manage these lifetimes automatically and which provide better ergonomics
in many other respects. As such, the plain-data approach would at best make raw
handles marginally more ergonomic for relatively uncommon use cases. This would
be a small benefit, and may even be a downside, if it ends up encouraging people
to write code that works with raw handles when they don't need to.

The plain-data approach also wouldn't need any code changes in any crates. The
I/O safety approach will require changes to Rust code in crates such as
[`socket2`], [`nix`], and [`mio`] which have APIs involving [`AsRawFd`] and
[`RawFd`], though the changes can be made gradually across the ecosystem rather
than all at once.

And, the plain-data approach would keep the scope of `unsafe` limited to just
memory safety and undefined behavior. Rust has drawn a careful line here and
resisted using `unsafe` for describing arbitrary hazards. However, raw handles
are like raw pointers into a separate address space; they can dangle or be
computed in bogus ways. I/O safety is similar to memory safety, both in terms
of addressing spooky-action-at-a-distance, and in terms of ownership being the
main concept for robust abstractions, so it's natural to use similar safety
concepts.

## New types for `RawFd`/`RawHandle`/`RawSocket`

Some comments on [rust-lang/rust#76969] suggest introducing new wrappers
around the raw handles. Completely closing the safety loophole would also
require designing new traits, since `AsRaw*` doesn't have a way to limit the
lifetime of its return value. This RFC doesn't rule this out, but it would be a
bigger change.

## I/O safety but not `IoSafe`

The I/O safety concept doesn't depend on `IoSafe` being in `std`. Crates could
continue to use [`unsafe_io::OwnsRaw`], though that does involve adding a
dependency.

## Define `IoSafe` in terms of the object, not the reference

The [reference-level-explanation] explains `IoSafe + AsRawFd` as returning a
handle valid to use for "the duration of the `&self` reference". This makes it
similar to borrowing a reference to the handle, though it still uses a raw
type which doesn't enforce the borrowing rules.

An alternative would be to define it in terms of the underlying object. Since
it returns raw types, arguably it would be better to make it work more like
`slice::as_ptr` and other functions which return raw pointers that aren't
connected to reference lifetimes. If the concept of borrowing is desired, new
types could be introduced, with better ergonomics, in a separate proposal.

# Prior art
[prior-art]: #prior-art

Most memory-safe programming languages have safe abstractions around raw
handles. Most often, they simply avoid exposing the raw handles altogether,
such as in [C#], [Java], and others. Making it `unsafe` to perform I/O through
a given raw handle would let safe Rust have the same guarantees as those
effectively provided by such languages.

The `std::io::IoSafe` trait comes from [`unsafe_io::OwnsRaw`], and experience
with this trait, including in some production use cases, has shaped this RFC.

[C#]: https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0
[Java]: https://docs.oracle.com/javase/7/docs/api/java/io/File.html?is-external=true

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Formalizing ownership

This RFC doesn't define a formal model for raw handle ownership and lifetimes.
The rules for raw handles in this RFC are vague about their identity. What does
it mean for a resource lifetime to be associated with a handle if the handle is
just an integer type? Do all integer types with the same value share that
association?

The Rust [reference] defines undefined behavior for memory in terms of
[LLVM's pointer aliasing rules]; I/O could conceivably need a similar concept
of handle aliasing rules. This doesn't seem necessary for present practical
needs, but it could be explored in the future.

[reference]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html

# Future possibilities
[future-possibilities]: #future-possibilities

Some possible future ideas that could build on this RFC include:

 - New wrapper types around `RawFd`/`RawHandle`/`RawSocket`, to improve the
   ergonomics of some common use cases. Such types may also provide portability
   features as well, abstracting over some of the `Fd`/`Handle`/`Socket`
   differences between platforms.

 - Higher-level abstractions built on `IoSafe`. Features like
   [`from_filelike`] and others in [`unsafe-io`] eliminate the need for
   `unsafe` in user code in some common use cases. [`posish`] uses this to
   provide safe interfaces for POSIX-like functionality without having `unsafe`
   in user code, such as in [this wrapper around `posix_fadvise`].

 - Clippy lints warning about common I/O-unsafe patterns.

 - A formal model of ownership for raw handles. One could even imagine
   extending Miri to catch "use after close" and "use of bogus computed handle"
   bugs.

 - A fine-grained capability-based security model for Rust, built on the fact
   that, with this new guarantee, the high-level wrappers around raw handles
   are unforgeable in safe Rust.

[`from_filelike`]: https://docs.rs/unsafe-io/0.6.2/unsafe_io/trait.FromUnsafeFile.html#method.from_filelike
[this wrapper around `posix_fadvise`]: https://docs.rs/posish/0.6.1/posish/fs/fn.fadvise.html

# Thanks
[thanks]: #thanks

Thanks to Ralf Jung ([@RalfJung]) for leading me to my current understanding
of this topic, and for encouraging and reviewing early drafts of this RFC!

[@RalfJung]: https://github.com/RalfJung
[`File`]: https://doc.rust-lang.org/stable/std/fs/struct.File.html
[`TcpStream`]: https://doc.rust-lang.org/stable/std/net/struct.TcpStream.html
[`FromRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html
[`FromRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html
[`FromRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html
[`AsRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.AsRawFd.html
[`AsRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.AsRawHandle.html
[`AsRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.AsRawSocket.html
[`IntoRawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.IntoRawFd.html
[`IntoRawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.IntoRawHandle.html
[`IntoRawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.IntoRawSocket.html
[`RawFd`]: https://doc.rust-lang.org/stable/std/os/unix/io/type.RawFd.html
[`RawHandle`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawHandle.html
[`RawSocket`]: https://doc.rust-lang.org/stable/std/os/windows/io/type.RawSocket.html
[`FromRawFd::from_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html#tymethod.from_raw_fd
[`from_raw_fd`]: https://doc.rust-lang.org/stable/std/os/unix/io/trait.FromRawFd.html#tymethod.from_raw_fd
[`from_raw_handle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html#tymethod.from_raw_handle
[`from_raw_socket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html#tymethod.from_raw_socket
[`SockRef::from`]: https://docs.rs/socket2/0.4.0/socket2/struct.SockRef.html#method.from
[`unsafe_io::OwnsRaw`]: https://docs.rs/unsafe-io/0.6.2/unsafe_io/trait.OwnsRaw.html
[LLVM's pointer aliasing rules]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules
[`nix`]: https://crates.io/crates/nix
[`mio`]: https://crates.io/crates/mio
[`socket2`]: https://crates.io/crates/socket2
[`unsafe-io`]: https://crates.io/crates/unsafe-io
[`posish`]: https://crates.io/crates/posish
[rust-lang/rust#76969]: https://github.com/rust-lang/rust/pull/76969
[`memfd_create`]: https://man7.org/linux/man-pages/man2/memfd_create.2.html