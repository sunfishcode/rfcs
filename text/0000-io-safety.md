- Feature Name: `io_safety`
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Close a hole in encapsulation boundaries in Rust by providing a way for users
of `AsRawFd`, `IntoRawFd`, and related traits to obtain useful guarantees about
their raw resource handles.

[rust-lang/rust#76969] makes `RawFd` implement `AsRawFd` and `IntoRawFd`. As
the comments note, the seeming unfortunate consequences here aren't new, and
come from the existing the existing types and traits in `std`. This RFC
proposes a new solution, introducing a new `OwnsRaw` trait and a concept of
I/O safety, building on and providing an explanation for the existing
`from_raw_fd` function being unsafe.

[rust-lang/rust#76969]: https://github.com/rust-lang/rust/pull/76969

# Motivation
[motivation]: #motivation

Rust's standard library today almost provides a concept called *I/O safety*,
a property that would guarantee that if one piece of code holds a resource
handle privately, other code in the program cannot access it.
[`FromRawFd::from_raw_fd`] today is unsafe, which prevents users from doing
things like `File::from_raw_fd(7)`, in safe Rust, to do I/O a file descriptor
which might otherwise be encapsulated.

However, there's a loophole. A somewhat common pattern in library APIs is to
use [`AsRawFd`] and [`IntoRawFd`] to accept handles to do I/O operations on,
like this:

```rust
pub fn do_something<FD: AsRawFd>(input: &FD) -> io::Result<()> {
    syscall(input.as_raw_fd())
}
```

`AsRawFd` doesn't put any restrictions on the return value of `as_raw_fd`
though, so the code above defines a function that can do I/O on arbitrary
`RawFd` values requiring the caller to use an `unsafe`. One can even write
`do_something(&7)`, since [`RawFd`] itself implements `AsRawFd`.

This can break encapsulation boundaries by creating aliases to resource
handles held privately elsewhere in the program, and can cause
[spooky action at a distance].

One solution which has been discussed for this is to wrap `RawFd` in a newtype.
On one hand, introducing a new type and getting users to switch to new types
and new traits would be much more disruptive. This RFC doesn't rule out doing
this in the future though.

This RFC introduces a new trait, `OwnsRaw`, which is designed to work with the
existing `std` types and traits instead of changing or replacing them, and
which provides a more gradual path to eventually closing the I/O safety
loophole. And it answers a question that has come up a [few] [times] about why
`from_raw_fd` is unsafe.

The expected outcomes are:

 - A new trait, `std::io::OwnsRaw`.
 - A new section in the ["Keyword unsafe"] page in the
   [Rust `std` reference documentation] documenting the I/O safety concept.
 - Revised documentation comments for [`FromRawFd::from_raw_fd`],
   [`FromRawHandle::from_raw_handle`], and [`FromRawSocket::from_raw_socket`],
   explaining why they're unsafe in terms of I/O safety.

[few]: https://github.com/rust-lang/rust/issues/72175
[times]: https://users.rust-lang.org/t/why-is-fromrawfd-unsafe/39670
[Rust `std` reference documentation]: https://doc.rust-lang.org/std/keyword.unsafe.html
[spooky action at a distance]: https://en.wikipedia.org/wiki/Action_at_a_distance_(computer_programming)

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The I/O Safety Concept

Rust's standard library has low-level types, [`RawFd`] on Unix-like platforms,
and [`RawHandle`]/[`RawSocket`] on Windows, which represent resource handles
provided by the OS. These don't provide any behavior on their own, and just
represent identifiers which can be passed to low-level OS APIs.

These raw resource handles can be thought of as raw pointers. They have similar
hazards; the consequences of using an unintentionally aliased raw resource
handle could include corrupted output or silently lost input data. It could
also mean that code in one crate could accidentally corrupt or observe private
data in another crate. Protection from these hazards is called *I/O safety*.

Rust's standard library also has high-level types such as [`File`] and
[`TcpStream`] which are wrappers around these low-level OS resource handles,
providing high-level interfaces to OS APIs.

These high-level types also implement the traits [`FromRawFd`] on Unix-like
platforms, and [`FromRawHandle`]/[`FromRawSocket`] on Windows, which provide
functions which wrap a low-level value to produce a high-level value. These
functions are unsafe, since they're unable to guarantee I/O safety. The type
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

I/O safety is new as an explicit concept, but it reflects existing common
practices which most Rust code is already following. For example, Rust's `std`
will require no changes beyond the introduction of a new trait and new impls
for it, because all the existing code already has the required properties.
Initially, not all of the Rust ecosystem will support I/O safety though, and
it's expected that adoption will be incremental.

## The `OwnsRaw` trait

These high-level types also implement the traits [`AsRawFd`]/[`IntoRawFd`] on
Unix-like platforms and
[`AsRawHandle`]/[`AsRawSocket`]/[`IntoRawHandle`]/[`IntoRawSocket`] on Windows,
which provide ways to obtain the low-level value contained in a high-level
value. Among other uses, these are used in APIs which wish to accept any type
which contains a raw handle value, such as in the `do_something` example in the
[motivation].

`AsRaw*` and `IntoRaw*` don't make any guarantees, so the example as written
doesn't guarantee I/O safety. `std::io::OwnsRaw` would be a new unsafe trait,
declared like this:

```rust
pub unsafe trait OwnsRaw {}
```

There are no required functions, so implementing it just takes one line,
plus comments:

```rust
/// # Safety
///
/// `MyType` wraps a `std::fs::File` which handles the low-level details, and
/// doesn't have a way to reassign or independently drop it.
unsafe impl OwnsRaw for MyType {}
```

It does require `unsafe`, to require the code to explicitly commit to
upholding I/O safety. With `OwnsRaw`, the `do_something` example should
simply add a `+ OwnsRaw`, like this, to provide I/O safety:

```rust
pub fn do_something<FD: AsRawFd + OwnsRaw>(input: &FD) -> io::Result<()> {
    syscall(input.as_raw_fd())
}
```

## Incremental adoption

I/O safety and `OwnsRaw` wouldn't need to be adopted all at once. If this RFC is
accepted, the sequence of events might look like this:

 - `OwnsRaw` is added to `std`, along with impls for all the relevant `std`
   types. This would be a backwards-compatible change.

 - After that, crates could implement `OwnsRaw` for their own types. These
   changes would be small and semver-compatible, so this wouldn't need any
   special coordination.

 - Once the standard library and enough popular crates utilize `OwnsRaw`,
   crates could start to add `+ OwnsRaw` bounds (or adding `unsafe`), at their
   own pace. These would be semver-incompatible changes, though most users of
   APIs adding `+ OwnsRaw` wouldn't be expected to need any changes.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## The I/O Safety Concept

In addition to the Rust language's memory safety, Rust's standard library also
guarantees I/O safety. An I/O operation is *valid* if the raw handles
([`RawFd`], [`RawHandle`], and [`RawSocket`]) it operates on are values
explicitly returned from the OS, and the operation occurs within the lifetime
the OS associates with them. Rust code has *I/O safety* if it's not possible
for that code to cause invalid I/O operations.

While Some OS's document their file descriptor allocation algorithms, a handle
value predicted from these algorithms isn't "explicitly returned from the OS".

Functions accepting arbitrary raw I/O handle values ([`RawFd`], [`RawHandle`],
or [`RawSocket`]) should be marked `unsafe` if they can lead to any I/O being
performed on those handles through safe APIs.

Functions accepting types which implement [`AsRawFd`]/[`IntoRawFd`] on
Unix-like platforms and
[`AsRawHandle`]/[`AsRawSocket`]/[`IntoRawHandle`]/[`IntoRawSocket`] on Windows
should add a `+ OwnsRaw` bound if they do I/O with the returned raw handle.

## The `OwnsRaw` trait.

Types implementing the `OwnsRaw` trait guarantee that they uphold I/O safety,
as described above. They must not make it possible to write a safe function
which can perform invalid I/O operations, and:

 - A type implementing `OwnsRaw` can only be constructed from a raw resource
   handle via an unsafe function, such as [`FromRawFd::from_raw_fd`].
 - A type implementing `AsRaw* + OwnsRaw` means its `as_raw_*` function returns a
   handle which is valid to use as long as the object passed to `self` is live.
   Such types may not have methods to close or reassign the handle without
   dropping the whole object.
 - A type implementing `IntoRaw* + OwnsRaw` means its `into_raw_*` function
   returns a handle which is valid to use at the point of the call.

`OwnsRaw` is implemented for all standard library types which implement
`AsRawFd` except for `RawFd` itself.

["Keyword unsafe"]: https://doc.rust-lang.org/stable/std/keyword.unsafe.html

# Drawbacks
[drawbacks]: #drawbacks

Drawbacks include:

 - Crates with APIs that use file descriptors, such as [`nix`], would need to
   switch to types which implement `AsRawFd + OwnsRaw`, use crates which
   provide equivalent mechanisms for doing this such as [`unsafe-io`], or
   change such functions to be unsafe.
 - Crates using `AsRawFd` or `IntoRawFd` to accept "any file-like type" or
   "any socket-like type", such as [`socket2`]'s [`SockRef::from`], would need
   to either add a `+ OwnsRaw` bound or make these functions unsafe.
 - This would rule out adding safe versions of `FromRawFd`, `FromRawHandle`,
   and `FromRawSocket`, which would be convenient for some use cases.
 - This explicitly extends the scope of `unsafe` to something outside of just
   memory safety and undefined behavior.

[`nix`]: https://crates.io/crates/nix
[`socket2`]: https://crates.io/crates/socket2

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## Handles as plain data

The main alternative would be to say that raw resource handles are plain data,
with no concept of I/O safety and no inherent relationship to OS resource
lifetimes. On Unix-like platforms at least, this wouldn't ever lead to memory
unsafety or undefined behavior.

However, most Rust code doesn't interact with raw resource handles directly.
This is a good thing, independently of this RFC, because resources ultimately
do have lifetimes, so most Rust code will always be better off using
higher-level types which manage these lifetimes automatically and which provide
better ergonomics in many other respects. As such, the plain-data approach
would at best making raw resource handles marginally more ergonomic for
uncommon use cases. This would be a small benefit, and may even be a downside,
if it ends up encouraging people to write code that works with raw resource
handles when they don't need to.

Another benefit of the plain-data approach would be that it wouldn't involve
any code changes in any crates. The I/O safety approach will require changes to
Rust code in crates such as [`socket2`] and [`nix`] which have APIs involving
[`AsRawFd`] and [`RawFd`]. However, the changes for I/O safety can be made
incrementally across the ecosystem rather than needing to happen all at once.

Another benefit of the plain-data approach would be that it keeps the scope of
`unsafe` limited to just memory safety and undefined behavior. Rust has often
drawn a careful line here and resisted using `unsafe` for describing arbitrary
hazards. However, raw handles are much like raw pointers into a separate
address space; they can dangle or be computed in invalid ways. I/O safety has
much in common with memory safety, both in terms of addressing
spooky-action-at-a-distance, and in terms of ownership being the main concept
for robust abstractions, so it's fairly natural to use similar safety concepts.

## Just improve the documentation

On a practical level, the problems addressed here aren't urgent for most Rust
users today, so Rust could arguably get by just by adding more documentation.

However, this would leave the `AsRawFd` loophole described in the [motivation]
open, leaving Rust without a way to fully encapsulate I/O.

## Newtypes for `RawFd`/`RawHandle`/`RawSocket`

This was suggested in [rust-lang/rust#76969], and this RFC doesn't rule out
doing something like this in the future, but it would be a much bigger change,
as it'd involve existing code migrating to using different types and traits.

## I/O safety but not `OwnsRaw`

`OwnsRaw` could be removed from this RFC. The main downsides would be that
we'd either have to recommend people who'd otherwise use it to add a
dependency on [`unsafe-io`] to use its `OwnsRaw`, or to use more `unsafe`.

# Prior art
[prior-art]: #prior-art

Most memory-safe programming languages have safe abstractions around raw
resource handles. Most often, they simply avoid exposing the raw resource
values altogether, such as in [C#], [Java], and others. Making it `unsafe` to
perform I/O through a given raw resource handle value would let safe Rust have
the same guarantees as those effectively provided by such languages.

The `std::io::OwnsRaw` trait proposed here comes from [`unsafe_io::OwnsRaw`],
and experience with this trait, including in some production use cases, has
shaped this proposal.

[C#]: https://docs.microsoft.com/en-us/dotnet/api/system.io.file?view=net-5.0
[Java]: https://docs.oracle.com/javase/7/docs/api/java/io/File.html?is-external=true

# Unresolved questions
[unresolved-questions]: #unresolved-questions

## Naming things...

Is there a better name for the `OwnsRaw` trait?

## Formalizing ownership

This RFC doesn't define a formal model for resource handle ownership and
lifetimes. The rules for raw handle values in this proposal are vague about the
identity such values. What does it mean for a resource lifetime to be
associated with a handle if the handle is just an integer type? Do all integer
types with the same value share that association?

The Rust [reference] defines undefined behavior for memory in terms of
[LLVM's pointer aliasing rules]; I/O could conceivably need a similar concept of
handle aliasing rules. This doesn't seem necessary for present practical needs,
but it could be explored later if there's a need for greater precision.

[reference]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[LLVM's pointer aliasing rules]: http://llvm.org/docs/LangRef.html#pointer-aliasing-rules

# Future possibilities
[future-possibilities]: #future-possibilities

Some possible future ideas that could build on this solution might include:

 - It may make sense to introduce new wrapper types around
   `RawFd`/`RawHandle`/`RawSocket`, to improve the ergonomics of some of the
   common use cases, however that can be explored separately. Such types may
   also provide portability features as well, abstracting over some of the
   `Fd`/`Handle`/`Socket` differences between platforms.

 - `OwnsRaw` enables higher-level abstractions. It may be interesting to
   explore adding features like [`from_filelike`] or other functions function
   in the [`unsafe-io`] crate to the standard library, as they eliminate the
   need for `unsafe` in user code in several common use cases.

   This is used in the [`posish`] crate to provide safe interfaces for
   POSIX-like functionality without having `unsafe` in user code, such as in
   [this wrapper around `posix_fadvise`].

 - A formal model of ownership for raw resource handles.

 - With a proper model of ownership for raw resource handles, one could imagine
   extending Miri to catch "use after close" and "use of invalidly computed
   handle" bugs.

 - A fine-grained capability-based security model for Rust, built on the
   fact that, with this new guarantee, the high-level wrappers around raw
   resource handles are unforgeable in safe Rust.

[`unsafe-io`]: https://crates.io/crates/unsafe-io
[`posish`]: https://crates.io/crates/posish
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
[`FromRawHandle::from_raw_handle`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawHandle.html#tymethod.from_raw_handle
[`FromRawSocket::from_raw_socket`]: https://doc.rust-lang.org/stable/std/os/windows/io/trait.FromRawSocket.html#tymethod.from_raw_socket
[`SockRef::from`]: https://docs.rs/socket2/0.4.0/socket2/struct.SockRef.html#method.from
[`unsafe_io::OwnsRaw`]: https://docs.rs/unsafe-io/0.6.2/unsafe_io/trait.OwnsRaw.html
