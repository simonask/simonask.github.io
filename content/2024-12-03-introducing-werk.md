+++
title = "Introducing Werk"
tags = ["werk", "rust"]
slug = "introducing-werk"
template = "blog-page.html"
[extra]
toc = true
draft = true
+++

Introducing Werk 💅
==================

`werk` is a command runner, similar to `just`, as well as a simplistic build
system, similar to `make`.

This is a general overview. For a more detailed explanation of syntax and
behavior, please consult the project's
[README][].

The source code for `werk` is [available on
GitHub](https://github.com/simonask/werk).

Check out [the examples](https://github.com/simonask/werk/tree/main/examples).

<blockquote class="caution">
<strong>Caution:</strong>
<code>werk</code> is alpha software. Some features may be buggy or missing. It may
eat your code. Use at your own risk. Commit everything before trying it out. Consider
trying it out in a virtual machine first. Don't blame me if you didn't.
</blockquote>

Just immediately, Why?!
------------------------

The motivation for `werk` is that `make` is annoying. It's a very sophisticated
tool that solves a bunch of problems I don't have, and doesn't solve many of the
problems I do have.

My personal use case - highly specific, but far from unique: 

- I'm building a video game.
- Some of the assets (WASM plugins) are built by Cargo.
- Other assets (shaders) are built by `glslc` and `slangc`. Shader compilation
  is slow, so precise dependency tracking (depfiles) is essential.
- The assets for the game are built "offline" into a custom asset pack format.
- Assets, including WASM code, can be hot-reloaded by the engine.
- Windows support is a hard requirement.
- When it doesn't do what I want, I need to be able to understand why.

Here's an incomplete list of problems I have with `make`:

- I want to use file names and paths that contain whitespace.
- I want build rules and "housekeeping" commands to not interact in surprising
  and bad ways *(he's a big fat `.PHONY`!)*.
- I want to work natively on Windows, as well as Linux and macOS. You can get
  GNU Make to sort of work using layers of emulation, but fundamentally it's a
  bad experience. Specifically, Makefiles typically rely on the presence of
  external POSIX utilities, like `sed`, `awk`, and so on, and all of these
  interact in bad ways with Windows filesystems.
- I want to put build artifacts in a separate directory.
- I want to use glob patterns in my build scripts (reliably), and I want them to
  be `.gitignore`-sensitive.
- I want to be able to diagnose my build process. The output of `make -d` is
  unreasonable.

Here's a 100% exhaustive list of problems I have with `just` (seriously, it's
great, but):

- It can't build things, only run commands.
- When you do anything fancy you generally need a shell, making `Justfile`s
  non-portable.

Here's a loose collection of problems I have with other purportedly
cross-platform tools:

- `ninja`: Too low-level, not nice to write by hand, very specialized for C/C++.
- `scons`: Very clunky in my opinion, annoying Python runtime dependency.
- `rake`: Ruby straight up does not work on Windows.
- `cargo xtask`: Only runs commands ("workflows"), similar to `just`.
- `cargo script`: Solves a different problem.
- `cmake`: I'm too traumatized already.
- All the Java tools (`gradle`, `maven`, `bazel`): I'm sorry, I can't.

Installation
------------

The only supported distribution method for `werk` is cloning the repository from
GitHub and building it from source. If this is a problem for you, `werk` probably
isn't a tool for you.

The only system dependencies are `rustc` and `cargo`.

```
$ git clone https://github.com/simonask/werk.git
$ cd werk
$ cargo build --release

# Executable is available at ./target/release/werk(.exe).
```

Design
======

`werk` parses a TOML file containing "recipes", determines what the user wants
to build, what the dependencies are, and runs programs to rebuild any outdated
build products in the right order.

> **Why TOML?** I don't want to write an LSP server. Who has the time.

> **Why not YAML?** I actually like YAML, and it would be easy to add support,
> but one crucial difference between the two is that YAML does not support
> "reopening" a mapping, so commands and recipes could not be expressed as
> naturally between each other.

> **Why not JSON or XML?** Neither is human-readable. Change my mind, I dare
> you to try.

Example
-------

Here's a minimal `werk.toml` to build a C program:

```toml
[global]
cc.which = "clang"
ld = "{cc}"

[build.'%.o']
in = "{%}.c"
depfile = "{%}.c.d"
command = "{cc} -c -o <out> <in>"

[build.'%.c.d']
in = "{%}.c"
command = "{cc} -MM -MT <in> -MF <out> <in>"

[build.'example{EXE_SUFFIX}']
in = { glob = "*.c", then = "{:.c=.o}" }
command = "{cc} <in*> -o <out>"

[command.build]
build = ["example{EXE_SUFFIX}"]
post-message = "Build complete!"
```

It's a regular TOML document, except that `werk` also parses string values
containing interpolations `{...}` or `<...>`.

*Example run:*

```
$ werk build
[OK] /foo.o
[OK] /main.o
[OK] /example.exe
[OK] build
[info] Build complete!
```

Outdatedness / Staleness
------------------------

The heart and soul of any build system. Targets are considered outdated under a
number of conditions:

1. **File modification time.** If an artifact is older than its prerequisites,
   it will be rebuilt.
2. **Glob patterns.** If a target takes the result of a glob operation (like
   `src/*.c`) as its input, `werk` detects when the result changed (including if
   files were deleted!), and rebuilds the target(s) that depend on the result.
3. **Environment variables.** If a target depends on the value of an environment
   variable, it will be considered outdated if the value changes between runs.
4. **`which` lookup.** If a target recipe uses a program that is found via the
   `PATH` environment variable, the target will be considered outdated if the
   path to the program changes.

Tracking glob results and environment variables requires some knowledge, other
than filesystem timestamps, to be transferred between runs. Werk writes a
`.werk-cache` file to its output directory containing only enough information to
determine whether things changed. Environment variables are hashed, so your
secrets should be safe, but be careful.

Werkspace & Output Directory
----------------------------

"Workspace" refers to the directory containing `werk.toml` and its
subdirectories, but not including any file or directory mentioned by
`.gitignore`, and not including the output directory.

"Output directory" refers to the location of build artifacts. `werk` will only
ever write to files within this directory, and puts `.werk-cache` in its root.

If `werk` detects a file named `.werk-cache` in the workspace (i.e., it exists
within a directory not mentioned by `.gitignore`), that is a hard error for your
protection.

<blockquote class="caution">
<code>werk</code> can use the same output directory as other build tools, like
<code>cargo</code>, but it doesn't try to avoid overwriting files generated by
other tools. Do this with caution.<br /><br />
<strong>Note:</strong> In the case of <code>cargo</code>, using the same output
directory also means that <code>cargo clean</code> will remove all build artifacts
produced by <code>werk</code>, including <code>.werk-cache</code>.
</blockquote>

Filesystem
----------

Filesystem paths are just fundamentally not portable between Unix-like systems
and Windows. Let's be honest - Windows is very weird. Using backslash `\` as the
path separator is just the start. Some file names are outright banned. Bet
you didn't know that [Windows will yell at
you](https://learn.microsoft.com/en-us/windows/win32/fileio/naming-a-file) if
you try to read or create a file called, say, `com¹.txt` (not a footnote - the
superscript `¹` is illegal when preceded by `com`, `COM`, `cOm`, or `CoM`,
regardless of the file name extension).

So even though Windows also does support using `/` as the path separator (unless
the path is "canonical" - this rabbit hole goes deep!), the set of legal
filesystem paths is just different.

`werk` ensures portability of your build scripts by introducing abstract paths,
where:

- The path separator is always forward slash `/`.
- Paths can only contain valid UTF-8. Incomplete UTF-16 pairs on Windows, or
  abitrary byte sequences on Linux and macOS, will cause an error.
- File and directory names must be valid on all platforms, so creating
  `com¹.txt` on Linux or macOS will also fail.
- Paths are always relative to the workspace root (the directory containing
  `werk.toml`) or the output directory. Paths cannot refer to files outside of
  the workspace.
- Paths are unambiguous: `/foo/../bar.txt` will always be reliably resolved as
  `/bar.txt`.
- Any abstract path can be losslessly converted to a native OS filesystem path
  on all platforms.

All input files and build artifacts are represented by an abstract path. Glob
expressions produce a list of abstract paths.

This is well and good, but `werk`'s main purpose is to execute other programs to
produce artifacts. These programs expect native OS paths. This is solved during
string interpolation, where there is a difference between `"{path}"` and
`"<path>"`.

- `"{path}"` will interpolate the abstract path as-is, verbatim.
- `"<path>"` will interpret `path` as an abstract path, and convert it to an
  absolute native OS filesystem path within the workspace or the output
  directory, depending on whether it is generated by a recipe or exists in the
  workspace.

Running other programs
----------------------

Uniquely, `werk` does not pass commands through a native shell, like `sh` or
PowerShell. Instead, `werk` spawns processes directly in code (using
`tokio::process::Command`).

I think this is a good idea because:

- POSIX-like shells are not available on Windows, so build scripts relying on
  them are not portable.
- Shell semantics differ wildly between platforms. For example, glob patterns
  are expanded by the shell on POSIX, but on Windows they are passed literally
  as an argument to the process, so the program must perform its own glob
  expansion.
- Constructing a command line invocation that results in the same command in all
  shells is incredibly error-prone. For example, if an argument contains spaces
  it must be quoted, and if it contains quotation marks, those must then be
  escaped, and what's the precise syntax for that?
- Handling `PATH` resolution in-process (instead of delegating to a shell) means
  we can track outdatedness.

The drawback is that shell features, such as stdio piping, are not available.
That's also kind of by design - again, piping works differently on different
platforms. `werk` does not currently provide an alternative to piping between
processes, but may do so in a future version.

Depfiles
--------

Many build tools and compilers understand and produce "depfiles", usually with
the file extension `.d`. These are in the Makefile format, and are designed to
be `@include`d in a `Makefile`.

`werk` has native support for parsing depfiles, including depfiles generated by
build recipes, and automatically adds dependencies from depfiles to a target's
dependencies if the `depfile` key is present in the recipe. (See the example
above.)

Controlling side effects: I/O sandboxing
----------------------------------------

Testing something like `werk` is challenging because there are fundamentally
many moving parts, especially external programs and files on the hard drive.

At the same time, the "dry run" mode is an essential feature, which also tries
to guarantee that modifications to the filesystem do not occur.

`werk` allows switching out the interface to the "real" system, such that every
side effect (writing files, spawning child processes) can be eliminated at
runtime. This feature is used both by the `--dry-run` mode, but also to test all
of the internal logic of `werk` without requiring filesystem access.

<blockquote class="info">
<strong>Note:</strong> Side effects in <code>--dry-run</code> are considered bugs,
but <code>werk</code> is alpha software and probably contains bugs.
</blockquote>

Variables
=========

`werk` supports variables and expressions.

- Global variables are defined in the `[global]` section of `werk.toml`, and
  their values are available to all recipes. Additionally, their values can be
  overridden from the command line, using `werk --define key=value`.
- Recipe-local variables are defined in a recipe-local table, `vars`, and are
  evaluated before running any commands.
- Implicit variables while evaluating expressions.

In the example above, `cc` and `ld` are global variables, and `ld` has the same
value as `cc`.

Please consult the [README][] for more details.

Strings & Interpolation
=======================

Strings within `werk.toml` are interpreted according to a custom string
interpolation DSL, loosely inspired by format strings in Rust.

String interpolation is specially suited for shell commands. When building a
command-line string, `werk` automatically handles quoting and whitespace in the
way you might expect.

One particular feature of string interpolation is path-sensitivity.

- `"{var}"` interpolates the value of `var` verbatim.
- `"<var>"` interprets `var` as an abstract file path, and converts it to an
  absolute native filesystem path.
- Both flavors can apply operations during interpolation, such as replacing a
  file extension: `"{var:.c=.o}"`.
- When a variable is a list (such as a list of input files), it can be expanded
  as `{var*}`, which will produce a string with each element of the list
  separated by a space. If `var` contains other lists, it will be expanded
  recursively (flat map).
- When an interpolation operation is present along with a list expansion, the
  operation is applied (recursively) to each element of the list. Example: `"<var*:.c=.o>"`.

In string interpolations, some automatic variables are available, with mnemonics
based on Make for convenience. For example, `"{%}"` interpolates the stem of the
pattern that was matched in the current recipe, if any.

Please consult the [README][] for more details.

Expressions
===========

When a variable is a TOML table (inline or block-style), it will be interpreted
as an expression, depending on the keys that are present in the table.
Expressions may be composed, or derived from other variables.

Expression evaluation tracks "outdatedness" as well. If a recipe uses a variable
that is *derived from*, say, an environment variable that changed since the last
run, the target is considered outdated.

Examples of expressions:

- `{ glob = "pattern", ... }`: Apply the glob pattern to the workspace and get a
  list of matching files.
- `{ which = "program" }`: Determine the absolute native OS filesystem path to
  `program`.
- `{ env = "NAME" }`: Read environment variable.
- `{ shell = "command" }`: Run `command` and capture its output. Confusingly,
  this doesn't actually invoke the system's shell, just the program.
- `{ ..., match = { 'foo' = "...", 'bar' = "..." } }`: Pattern-match against an
  input and conditionally evaluate the result.
- `{ ..., patsubst = { pattern = "...", replacement = "..." } }`: Apply a
  pattern-substitution to the result of another expression.

Please consult the [README][] for more details.

Modes
=====

`werk` can be invoked in different modes to understand and diagnose what's
happening.

- `--dry-run`: Do not run any commands or write any files. Note that `--dry-run`
  impedes depfile support, because `werk` cannot build the depfiles, so
  dependencies found in those may be stale or nonexistent.
- `--explain`: For every target being rebuilt, explain in the terminal why
  `werk` decided it was outdated. The explanation includes *all* reasons that a
  target was outdated, including cached information about the environment. This
  option tries very hard to be as user-friendly as possible, while still giving
  a full picture.
- `--print-commands`: Print external command line invocations as they are being
  executed, or would have been executed in a dry-run.


Final Remarks
=============

<blockquote class="rainbow">
<strong>Important:</strong>
<code>werk</code> is an LGBTQIA+ program (yes, all of them). You can use it freely,
except if you personally are homophobic, biphobic, transphobic, or support any politician
or work for any organization that harms or opposes minority rights. Otherwise, we love
you and support you. 💖
</blockquote>

Implementing `werk` has been very fun. As you may have guessed, it's written in
Rust, and it leans into async/await with Tokio for managing concurrency.

I'm used to writing code where performance is a high priority, but for a tool
that is kind of fundamentally "heavy", in the sense that its purpose is to
invoke much more heavy-duty programs such as compilers, it doesn't make a lot of
sense to spend time on shaving off those microseconds. Joyfully, Rust was still
a very good choice here.

**How was Rust helpful?**

- We're running things in parallel, and Rust ensures that there are no
  synchronization bugs or race conditions, period.
- We're invoking external commands, so controlling errors and failure modes is
  important. We don't want to abruptly kill a process just because one of its
  siblings failed.

There's 3 uses of `unsafe` in the entire project, all trivial one-liners with
guaranteed semantics (transparent reference casting), all related to converting
between `werk_fs::Path` and `str`.

Shout-out to the dependencies that made things significantly easier, and their
authors: `which`, `globset`, `winnow`, `clap`, and not least, `tokio`.

[README]: https://github.com/simonask/werk/blob/main/README.md
