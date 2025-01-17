+++
title = "Introducing Werk"
tags = ["werk", "rust"]
slug = "introducing-werk"
template = "blog-page.html"
[extra]
toc = true
+++

Introducing Werk 💅
==================

[I made a thing.](https://github.com/simonask/werk)

`werk` is a simplistic build system, similar to `make`, and a command runner,
similar to `just`. It tries to be very easy to use, and religiously portable.

This is a general overview. For a more detailed explanation of syntax and
behavior, please consult the project's [README][].

The source code for `werk` is [available on GitHub](https://github.com/simonask/werk).

Check out [the book](https://simonask.github.io/werk) and [the
examples](https://github.com/simonask/werk/tree/main/examples).

<blockquote class="caution">
<strong>Caution:</strong>
<code>werk</code> is alpha software. Features may be buggy or missing. It may
eat your code. Use at your own risk. Commit everything before trying it out. Consider
trying it out in a virtual machine first. Don't blame me if you didn't.
</blockquote>

Just immediately, Why?!
------------------------

The motivation for `werk` is that `make` is annoying. Make is a very
sophisticated tool that solves a bunch of problems I don't have, and doesn't
solve many problems that I *do* have.

My personal use case - highly specific, but far from unique:

- I'm building a video game. Video games have a complex build process, because
  they contain both code and assets.
- Iteration speed is important.
- Some assets are code:
  - WASM plugins built by Cargo.
  - Shaders built by `glslc` and `slangc`. Shader compilation is slow, so
    precise dependency tracking (depfiles) is important.
- Native Windows support is a hard requirement.
- When it doesn't do what I want, I need to be able to understand why.

### Why not Make?

Here's an incomplete list of problems I have with `make`:

- I want to use file names and paths that contain spaces.
- I want build rules and "housekeeping" workflows to not interact in surprising
  and bad ways *([`.PHONY`](https://www.youtube.com/watch?v=ToQVoyWWluQ))*.
- I want to work natively on Windows, as well as Linux and macOS. You can get
  GNU Make to sort of work using layers of emulation, but fundamentally it's a
  bad experience. Specifically, Makefiles typically rely on the presence of
  external POSIX utilities, like `sed`, `awk`, and so on, and all of these
  interact in undesirable ways with Windows filesystems.
- I want to put build artifacts in a separate directory.
- I want to use glob patterns in my build scripts (reliably), and I want them to
  be `.gitignore`-sensitive, like all the other tools we use.
- In general, file modification times are just insufficient in determining
  outdatedness.
- I want variables, lists, strings, and other things we expect of contemporary
  languages.
- I want to be able to diagnose my build process. The output of `make -d` is
  unreasonable.

### Why not `just`?

Here's a 100% exhaustive list of problems I have with
[`just`](https://just.systems/man/en/):

- It can't build things, only run commands.
- When you do anything fancy you generally need a shell, making `Justfile`s
  non-portable.
- Other than that, it's great, seriously. Big fan.

### Why not $toolname?

Here's a loose collection of problems I have with other purportedly
cross-platform tools:

- `ninja`: Too low-level, not nice to write by hand, very specialized for C/C++.
- `scons`: Very clunky in my opinion, annoying Python runtime dependency.
- `rake`: Ruby straight up does not work on Windows.
- `cargo xtask`: Only runs commands ("workflows"), similar to `just`.
- `cargo script`: Solves a different problem.
- `cmake`: I'm too traumatized already.
- All the Java tools (`gradle`, `maven`, `bazel`): I'm sorry, I can't.

Overview
========

`werk` parses a Werkfile containing recipes, determines what the user wants to
build, what the dependencies are, and runs programs to rebuild any outdated
build products in the right order.

Consult [the book](https://simonask.github.io/werk) for many, many more details.

> **Why a new language?** I tried various things, including expression build
> rules declaratively in TOML. `werk` actually still supports this mode, mainly
> for testing. But it quickly becomes very unwieldy to express the kind of
> things you really want to express in build scripts.
>
> It sucks, I know, but I promise it's fairly pleasant and readable.

Example building C program
--------------------------

Here's a minimal Werkfile to build a C program. It showcases build recipes (with
depfiles), command recipes ("workflows"), and various forms of string
interpolation. See also [the language
reference](https://simonask.github.io/werk/language.html).

```werk
let cc = which "clang"
let ld = cc

build "%.o" {
  from "{%}.c"
  depfile "{%}.c.d"
  run "{cc} -c -o <out> <in>"
}

build "%.c.d" {
  from "{%}.c"
  run "{cc} -MM -MT <in> -MF <out> <in>"
}

build "example{EXE_SUFFIX}" {
  from glob "*.c" | map "{:.c=.o}"
  run "{ld} <in*> -o <out>"
}

task build {
  build "example{EXE_SUFFIX}"
  info "Build complete!"
}
```

*Example run:*

```text
$ werk build
[ ok ] /foo.o
[ ok ] /main.o
[ ok ] /example.exe
[ ok ] build
[info] Build complete!
```

Example building a Rust project with Cargo
------------------------------------------

```werk
let cargo = which "cargo"

build "debug/my-program{EXE_SUFFIX}" {
  # Werk understands depfiles emitted by Cargo
  depfile "debug/my-program.d"

  # This will only run if any of the source files discovered by Cargo
  # have changed since the last run.
  run "{cargo} --profile=dev -p my-program"
}

task clean {
  run "{cargo} clean"
}

task build {
  build "debug/my-program{EXE_SUFFIX}"
}
```

*Example run:*

```text
$ werk build
[ ok ] /debug/my-program.exe
[ ok ] build
```

Remarks
=======

`werk` is a side project. Contributions are welcome, but ultimately it is
designed to address my personal use cases. If you believe it should also address
your use case, please feel free to get in touch.

In defense of getting sidetracked
---------------------------------

Did I *have* to make `werk`? Not really. I could have made GNU Make work for my
purposes. In fact, I did for a long time. On top of that, I'm trying to deliver
a video game, one of the notoriously hardest thing to actually finish.

The most challenging thing about finishing a video game as a solo developer is
to stay motivated. There are many problems that are hard to solve, and that's a
big part of the fun, but it's also a constant pull on my creativity. That's both
exciting and incredibly draining.

Toughing it out and powering through does not work. What works is to take a
break, step away from the trenches, and let the creative juices stew in the
background for a while.

Doing little side projects that feel useful - and hopefully contribute a bit to
the broader ecosystem - is a distraction, but it's also a way to stay healthy
and productive, and combats the feeling of isolation.

Rust
----

As you may have guessed, `werk` is written in Rust, and it leans into
async/await for managing concurrency. It's no secret that Rust is my current
favorite language, and I use it for almost everything. I want to share why.

**How was Rust helpful?**

- We're running things in parallel, and Rust ensures that there are no
  synchronization bugs or race conditions, period.
- We're invoking external commands, so controlling errors and failure modes is
  important. We don't want to abruptly kill a process just because one of its
  siblings failed.

There's a couple of uses of `unsafe` in the entire project, all trivial
one-liners with guaranteed semantics (transparent reference casting), mostly
related to converting between `werk_fs::Path` and `str`.

Shout-out to the dependencies that made things significantly easier, and their
authors: `which`, `globset`, `winnow`, `clap`, and `smol`.

[README]: https://github.com/simonask/werk/blob/main/README.md
