+++
title = "Introducing Stringleton"
tags = ["stringleton", "rust"]
slug = "introducing-stringleton"
template = "blog-page.html"
[extra]
toc = false
+++

# Introducing Stringleton

[Stringleton](https://docs.rs/stringleton/latest/stringleton/) is a very fast
string interning library.

*String interning:* The technique of representing all strings which are equal by
a pointer or ID that is unique to the *contents* of that strings, such that O(n)
string equality check becomes a O(1) pointer equality check.

The defining characteristic of Stringleton is [the `sym!(...)`
macro](https://docs.rs/stringleton/latest/stringleton/macro.sym.html). It allows
the creation of interned strings (symbols) from literals in the code with
practically zero overhead.

## The current state of things

Normally, creating an interned string from a literal in code involves the
following steps:

1. Acquire a global lock on some symbol registry.
2. Determine if the string is already present in the registry.
3. If so, return the existing ID or pointer.
4. Otherwise, allocate the backing storage for the symbol and insert it in the
   registry.

All of this means that creating symbols in code is something you don't want to
do in inner loops, or you may want to avoid it if you are scared of "death by a
thousand cuts". In particular, the need to acquire a global lock or other
synchronization primitives every time is quite the impediment.

## How Stringleton does it

Instead, Stringleton uses a novel approach with some cooperation from the
linker:

1. Every call site of `sym!(...)` is statically registered at link time, using
   `linkme::distributed_slice`. This creates a small "symbol table" in the
   `.bss` section of the resulting binary.
2. When the process starts, all entries in the symbol table are "reconciled",
   effectively performing steps 2 through 4 of the above (using the `ctor`
   crate).
3. When actually reaching the `sym!(...)` call site, it compiles into a single
   memory load.

The real kicker here is that the memory load at the call site *does not even
need to be atomic.* The reason this is safe and sound is that static initializer
functions (`ctor`) are guaranteed to execute before `main()`, and therefore
before any threads can possibly have started. In other words, by the time
`main()` starts executing, the symbol table is already fully initialized, and it
is never modified again in the lifetime of the process.

The caveat is, of course, that using `sym!(...)` in another static initializer
function is *not* safe.

### Generated code

Consider this function:

```rust
fn get_symbol() -> Symbol {
    sym!("Hello, World!")
}
```

This compiles into a single load instruction. Using `cargo disasm` on x86-64
(Linux):

```asm
get_symbol:
  8bf0    mov  rax, qword ptr [rip + 0x52471]
  8bf7    ret
```

For the uninitiated, that's a plain non-atomic memory load. Most notably, there
are no function calls or locks acquired.

## Life before `main()`, a diatribe

The general assumption in Rust is that nothing happens before `main()`. This
isn't actually true, and it's worth considering what the nature is of that
assumption.

Static initializer functions (or static constructors) fall in a weird kind of
grey area when it comes to safety. The big question is: *Is safe code in
`main()` allowed to rely on the invariant that static initializers have run
before `main()`?*

The answer currently seems to be a resounding maybe. It would be a useful rule,
as this crate demonstrates, but at the same time, static constructors is a
linker feature that may not be guaranteed to exist everywhere. In C, they
require nonstandard attributes. In C++ they are guaranteed to execute before
`main()`, but esoteric configurations may forbid it.

Multiple things in the standard library do not work *inside* static
constructors, but some things do: `alloc` and atomic synchronization primitives
like `OnceLock`. It is unclear whether the things that don't work are guaranteed
to always fail in safe ways.

All `#[ctor]` static initializer functions should be fundamentally considered
`unsafe`, but not in the normal sense, because it isn't about the function's
preconditions (the function itself may be perfectly safe), but rather about
whether the function relies on the implicit precondition that static
initializers have run. In other words, the semantics are inverted: The function
may be unsafe to call in a static initializer, even though it only invokes safe
APIs, because those safe APIs may rely on other static initializers having
finished.

In effect, the "unsafe" part of a static initializer function is the fact that
it is being called before `main()`, and that any function may rely on static
initializers having actually completed.

Before Edition 2024, Rust had no way of expressing this inversion, but now we
have "unsafe attributes". An example of this is `no_mangle`, which is now
spelled `#[unsafe(no_mangle)]`. The reason it matters for `no_mangle` is that
defining a global symbol with a particular name may shadow other symbols with
the same name (on some platforms). It can effectively override some arbitrary
function, given the right linker flags, and that is deeply unsafe for obvious
reasons. On Linux and macOS, there are even official ways to do this: The
`LD_PRELOAD` environment variable explicitly allows overriding symbols from an
executable with symbols from a dynamic library, which is another example of a
thing that "happens before `main()`".

The `ctor` crate does not yet require `#[unsafe(ctor)]`, but it [hopefully
will](https://github.com/mmastrac/rust-ctor/issues/159).
