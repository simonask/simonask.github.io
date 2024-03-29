+++
title = "libyaml-safer"
tags = ["yaml", "rust"]
slug = "libyaml-safer"
template = "blog-page.html"
[extra]
toc = false
+++

Porting libyaml to Safe Rust: Some Thoughts
===========================================

The following is a loose collection of thoughts that came up while porting
[unsafe-libyaml] to safe, idiomatic Rust, which may be useful to someone else
who is considering "rewriting it in Rust", as the ancient journeyman prayer
goes.

_Some context:_ [YAML] is a commonly used markup language, similar to JSON, and
the primary way that Rust programmers interact with it is through [serde_yaml],
a serializer/deserializer implementation targeting YAML through the very popular
Serde framework, which is backed by an almost verbatim machine translation of
the C [libyaml] library into unsafe Rust, called [unsafe-libyaml]. The
translation is made with [c2rust], with only minor fixes applied manually.

In my case, I was in a situation where I needed to interact with YAML in a way
where [serde_yaml] came up short. Specifically I needed input location markers
outside of error messages to support diagnostics and debugging in the context of
a game, where the AI behavior trees are defined in YAML files.

But the unsafe libyaml API is actually pretty unwieldy to use from Rust, even
when translated to Rust code, because it behaves exactly like the C code, so it
is no different than calling out to an actual C library with FFI. So I - perhaps
foolishly, in a flash of manic youthful optimism - decided to embark on the
journey of porting [unsafe-libyaml] to safe Rust, function by function, line by
line. It took a little over a week.

The [result is available on GitHub](https://github.com/simonask/libyaml-safer).

**Update 2024-02-09:** libyaml-safer has now been [published to
crates.io](https://crates.io/crates/libyaml-safer) as well.

## What are we dealing with?

libyaml consists of two main components:

* YAML parser, which consumes a stream of bytes and produces "events"
  representing the structure of the document.
* YAML emitter, which consumes a stream of events and produces a stream of
  bytes.

The byte streams, both input and output, can be in different encodings, one of
UTF-8, UTF-16 little-endian, or UTF-16 big-endian. Most, but not all, Unicode
code points are supported - notably, control characters are not supported.

Events in libyaml are a linear representation of the logical structure of a
document, so higher level than parser tokens, but lower level than mappings,
sequences, and scalar values. Events are things like "begin mapping", "begin
mapping key", "scalar", etc.

Libyaml also contains a high-level abstraction, `yaml_document_t`, which can
represent any YAML document. The parser can load a stream of events into a
document, and the emitter can produce a stream of events from a document.

## Eldritch Horrors, or just C

[libyaml] is squarely a C library, and reading it from the point of view of a
Rust programmer, there is a beautiful insanity to it all. It hails from a
simpler time, where concepts like ownership, mutability, and the fabric of
reality were up for debate.

It isn't "wrong", just different. Unnecessarily difficult in many ways, but in
other ways refreshingly straightforward, so long as you can keep the cosmic
horror only just barely out of sight.

### String Theory

YAML is a human-readable format, and any parser for it must deal with a _lot_ of
strings. Thankfully, libyaml does the sane thing and defines its own string type
\- which almost every single C library in existence does, and even many larger
C++ libraries - and sometimes they don't even contain very many bugs. I swear
the history of the world would have looked very, very different if our venerable
forebears had extended their otherwise infinite benevolence to the inclusion of
a sane string type with the standard C library, but again I digress.

This is the string abstraction in [libyaml]:

```c
struct yaml_string_t {
    char* start;
    char* end;
    char* pointer; // <-- ???
};
```

The `start` and `end` fields seem straightforward: A pointer to the beginning of
the string and a pointer to one-past-the-end of the string. So far so good. But
what about `pointer`? Some places in the code it is used as the actual end of
the string (so `end` is actually the end of the allocated capacity of the
string), and in others it is used as the actual start of the string, so `start`
is the beginning of the underlying allocation, and `pointer` is used to iterate
over bytes or substrings of the string.

What's going on here?

It turns out that this data structure serves multiple different purposes
simultaneously: It is both `&str`, `String`, and `std::slice::Iter`, all in one.
Sometimes it is even the `std::str::Chars` iterator, which is used to iterate
over full UTF-8 codepoints within the string. The only way to determine which
abstraction it is currently emulating is to look at the context.

Thankfully - libyaml being a well-structured C project - there are plenty of
context clues. When the `STRING_DEL()` macro is being used, the caller assumes
ownership. When the `STRING_ASSIGN()` macro is being used, the string is being
borrowed and/or used as an iterator.

The interesting takeaway is this: For all of the different roles that
`yaml_string_t` takes on, it turns out there is an abstraction in the Rust
standard library that matches each use case precisely. At all points in the
libyaml source code where strings are used, the use case maps directly to one of
the _several_ many ways of dealing with strings in Rust. Phew!

_Joyful aside:_ There is a similar type in libyaml, called `yaml_buffer_t`. It
is slightly more complicated than strings, because it acts essentially as a ring
buffer, and it turned out to be almost very close to a `VecDeque` from the
standard library. Hooray!

## Self-referential types - Oh my!

YAML syntax can represent the same thing in multiple ways. For example,
sequences and mappings can be represented in both "flow style" and "block
style", and strings can be single-quoted, double-quoted, unquoted, block-quoted,
etc.

```yaml
flow_style_mapping: { a: "Hello", b: "World" },
block_style_mapping:
  a: Hello
  b: World
flow_style_sequence: [1, 2, 3]
block_style_sequence:
- 1
- 2
- 3
```

The libyaml emitter wants to choose both nice and correct formatting for the
output. Not just for the aesthetic, but also because some things actually
require specific styles - for example, escape sequences are only supported in
double-quoted strings, and some Unicode characters need to be represented as
escape sequences.

The emitter works by pushing "events" to a queue. Events are things like "begin
mapping", "begin mapping key", "begin mapping value", "end mapping", and so on,
and the emitter writes to the output when it has enough information to determine
the appropriate style.

This is implemented in libyaml as an "analysis" step that happens for each event
before the corresponding YAML is written to the output.

```c,linenos,linenostart=1700
static int
yaml_emitter_analyze_event(yaml_emitter_t *emitter,
        yaml_event_t *event)
{
    emitter->anchor_data.anchor = NULL;
    emitter->anchor_data.anchor_length = 0;
    emitter->tag_data.handle = NULL;
    emitter->tag_data.handle_length = 0;
    emitter->tag_data.suffix = NULL;
    emitter->tag_data.suffix_length = 0;
    emitter->scalar_data.value = NULL;
    emitter->scalar_data.length = 0;

    switch (event->type)
    {
        // ...
    }
    // ...
}
```

The analysis step populates the `emitter->anchor_data`, `emitter->tag_data`, and
`emitter->scalar_data` fields depending on the event type. Those fields that can
be `NULL` are C strings, and they refer to the strings contained within the
`event`. But the `event` comes from a queue owned by `yaml_emitter_t`. This is
great design in the C library, because it makes the analysis step extremely
cheap - no strings are duplicated, only "borrowed".

So how do we express the analysis fields in safe Rust? Blindly porting the C
approach to Rust would result in `yaml_emitter_t` becoming self-referential,
since the analysis pointers would either point into the event queue or into an
event that was just popped from the queue, making it impossible to represent
that lifetime on `yaml_emitter_t` itself.

**Key Insight:** Every time `yaml_emitter_analyze_event()` is called, it resets
any previous analysis results. It is called exactly once per event. Aha! What if
the results of the analysis was a return value from
`yaml_emitter_analyze_event()`, instead of modifying fields on the emitter?

```rust
#[derive(Default)]
struct Analysis<'a> {
    pub anchor: Option<AnchorAnalysis<'a>>,
    pub tag: Option<TagAnalysis<'a>>,
    pub scalar: Option<ScalarAnalysis<'a>>,
}

struct AnchorAnalysis<'a> {
    pub anchor: &'a str,
    pub alias: bool,
}

struct TagAnalysis<'a> {
    pub handle: &'a str,
    pub suffix: &'a str,
}

struct ScalarAnalysis<'a> {
    /// The scalar value.
    pub value: &'a str,
    /// Does the scalar contain line breaks?
    pub multiline: bool,
    /// Can the scalar be expessed in the flow plain style?
    pub flow_plain_allowed: bool,
    /// Can the scalar be expressed in the block plain style?
    pub block_plain_allowed: bool,
    /// Can the scalar be expressed in the single quoted style?
    pub single_quoted_allowed: bool,
    /// Can the scalar be expressed in the literal or folded styles?
    pub block_allowed: bool,
    /// The output style.
    pub style: ScalarStyle,
}

fn yaml_emitter_analyze_event<'a>(
    emitter: &yaml_emitter_t,
    event: &'a yaml_event_t,
) -> Analysis<'a> {
    match event.type_ {
        // ...
    }
}
```

Wonderful! Now the `Analysis<'a>` struct just needs to be passed into any
emitter function that needs the analysis as an extra parameter. Thankfully no
public API function on the emitter relies on analysis having happened
previously, so threading that argument through the emitter code is fairly
trivial, and guaranteed bug-free due to lifetime annotations, in the sense that
it is not possible for any part of the emitter to use the analysis of one event
during the emission of another event.

On the one hand, this change is something that we're forced to do by Rust,
because it would not be possible to make the borrow checker happy without it. On
the other hand, the Rust compiler actually nudges us to choose a better design
than the original.

Why is this beneficial? Why do we accept such blatant tyranny by the cruel and
cold-hearted borrow checker?

Well, see, reading the original source code for libyaml was hard. Upon seeing
the `anchor_data`, `tag_data`, and `scalar_data` fields of `yaml_emitter_t`, the
first thing any reader or maintainer has to do is figure out how those pointers
are used and when they are valid. This required a whole bunch of forensics,
which would have to happen again and again every time someone new looked at the
code, and "someone new" also includes "future you".

The original design from the C code could have been ported verbatim to Rust. But
it would require a lot of unsafe code, which would have made it very obvious
that some cognitive overhead was being incurred. By choosing a design that works
in safe Rust, that cognitive overhead has been moved to a statically verified
invariant that is plainly visible in the source code. The new signature of
`yaml_emitter_analyze_event` plainly documents that `Analysis<'a>` contains
references that are coming from a `yaml_event_t` with lifetime `'a`.

## Error handling

One of my goals with libyaml-safer was to provide near perfect error fidelity
compared with libyaml.

Mercifully, C does not have exceptions. They are easily among the top 20 of
humanity's mistakes, and don't even come for me because I will die on this hill.

Instead, C code usually reports errors by returning an error code, and this is
already a primitive version of what Rust does. The flow around error handling is
surprisingly similar between well-structured C code and idiomatic Rust code.
Rust has the "try" operator `?` to make things easier, and one of the important
affordances of [unsafe-libyaml] (Rust) over [libyaml] (C) is that integer return
codes are replaced with a bespoke `Success` struct. Replacing this with
[`Result`] turned out to be [pretty
trivial](https://github.com/simonask/libyaml-safer/commit/dc639b92b5754f58e8e8f7979449ba6aca54c3a6).

But wait a minute... We can't just use the `?` straight away. It checks if an
error occurred and returns early from the function if it did, but C does not
have RAII, so resources need to be released as well before returning. This is
the one remaining valid use case of `goto` in the C language, and it is often
actually the most sustainable way to deal with errors. More civilized languages
(such as Rust) don't even have this misfeature.

```c
void* stupid_c() {
    void* some_memory = malloc(1024);
    if (some_function(some_memory) != ERROR) {
      // ...
    } else {
      goto cleanup;
    }

    if (some_other_function(some_memory) != ERROR) {
      return some_memory;
    }

cleanup:
    // We're returning an error, so we need to free what we allocated.
    free(some_memory);
    return NULL;
}
```

Thus, in order to start leveraging the error handling features of Rust, all
things that own memory must first be converted to RAII objects, such as
`String`, `Vec`, `VecDeque`, etc.

After that, there is the problem of actually reporting the error.

Rust `Result`s are rich: They can contain an error value of your choice, so you
can provide the best possible message for the users, giving them all the context
they need. C does not have this option built in, and most libraries opt for
simply returning an error code, and then sometimes allowing the programmer to
obtain more information about the error after the fact.

In libyaml, this looks like so:

```c
struct yaml_parser_t {
    yaml_error_type_t error;
    const char* problem;
    yaml_mark_t problem_mark;
    const char* context;
    yaml_mark_t context_mark;
    // ...
}
```

When an error occurs, the relevant function (such as `yaml_parser_parse()`) sets
the error fields (`yaml_set_parser_error()`), and returns an integer indicating
that an error occurred. The user is then expected to access these fields to
determine what the error was and where in the input it occurred.

In Rust, we don't want a special "error" state for the parser. Ideally, we want
to be able to return a self-contained error instead, which can then have all the
usual niceties, like an implementation of `std::fmt::Display`.

Note also that the reported error is not necessarily a "parse error". Lots of
errors can occur that aren't typically considered syntax errors: I/O read
errors, invalid Unicode, tokenization errors, etc.

Let's try with some enums:

```rust
enum ParserError {
    Problem {
        problem: &'static str,
        problem_mark: Mark,
        context: &'static str,
        context_mark: Mark,
    }
    Scanner(ScannerError),
}

enum ScannerError {
    Problem {
        problem: &'static str,
        problem_mark: Mark,
        context: &'static str,
        context_mark: Mark,
    },
    Reader(ReaderError),
}

enum ReaderError {
    Problem {
        problem: &'static str,
        offset: usize,
        value: i32,
    },
    Io(std::io::Error),
}
```

So nice, perfect fidelity and high-quality error reporting. Except... On a
64-bit architecture, the size of the top-level `ParserError` enum is 88 bytes,
mostly due to the `Mark` type, which indicates a location in the input, and
which looks like this:

```rust
struct Mark {
    pub line: u64,
    pub column: u64,
    pub index: u64,
}
```

24 bytes, and each error typically needs 2 marks.

Why is the size a problem?

1. Even the minimal `Result<(), ParserError>` can never be returned in a register.
2. Every function that encounters a `Result<T, ParserError>` and wants to return
   `Result<U, ParserError>` needs to re-wrap the return value, no matter if an
   error occurred or not, leading to a lot of copying of large structs in stack
   memory.
3. While LLVM will [inline calls to `memcpy()` below 128
   bytes](https://github.com/rust-lang/rust/pull/64302#issuecomment-529840404),
   instructions to copy these values are still generated.

The parser makes a huge number of tiny calls, and each of them can fail. Every
time it needs a new token, it might also need a new character, meaning it might
also need another byte from the input buffer, meaning it might also need to
perform a `read()` operation from the input, which can fail with
`std::io::Error`, or with a Unicode decoding error, or with a tokenization
error, and finally a parser error.

All in all, a very large error type like this is... not great.

There are a couple of options for moving forward:

1. Mimic the libyaml style of errors, and set error codes on the parser object
   itself. This gives a much less ergonomic API, or would introduce a lifetime
   parameter on all return values that isn't very nice.
2. Codegolfing the above enums to reduce the size. Not much can be won without
   losing fidelity, but it could probably get down to about 64 bytes.
3. Wrap the error in a `Box`, reducing the size to only 8 bytes on 64-bit
   platforms. The heap allocation isn't great, but it would only happen in error
   scenarios, so it's trading some efficiency in the error path for a faster
   "happy" path.
4. Accept that this is our life now and move on.

All compromises, either in usability or performance.

I benchmarked options (3) and (4) against each other in the happy path, and
there wasn't much difference. But it still _feels wrong_, and committing to an
"inefficient" public API where the user can see the inner workings of the error
enums is a semver hazard, so for now I've chosen an API that supports both
implementations, and boxed (3) under the hood.

Zooming out, it's worth noting that
[`serde_yaml::Error`](https://docs.rs/serde_yaml/latest/src/serde_yaml/error.rs.html#12)
has actually chosen (3), a boxed error, which is an indication that I am not the
only person in the world in considering this an acceptable tradeoff.

## Freebies

Rust, being a younger language, has a much richer standard library. This means
that several things that are implemented manually in libyaml can be replaced
with standard facilities.

- UTF-8 and UTF-16 encoding/decoding. These aren't particularly complicated, but
  the Rust standard library is likely to be better optimized (for example, using
  SIMD when available).
- Rather than user-provided unsafe callbacks, input and output can be handled by
  the `std::io::Read` and `std::io::Write` traits. This gives users the ability
  to plug anything in there, such as tty streams, network streams, etc., with no
  extra effort.
- The parser requires some input buffering to perform correct decoding, but
  libyaml is practically forced to do this by itself, even when the input is
  already buffered by the user. By using the `std::io::BufRead` trait, an entire
  intermediate buffer was eliminated from the parser, meaning that if the data
  is already coming from an in-memory buffer, the unnecessary copy is elided.

## Testing

`unsafe-libyaml` is automatically tested against [the official YAML test
suite](https://github.com/yaml/yaml-test-suite). Having this comprehensive test
suite was an absolute necessity when porting to safe Rust, and I would not have
dared to embark on this endeavor without it.

For every change to the code, I was able to run the full test suite and verify
that everything was as it should be. And even though the code was ported with a
true dedication to matching the original semantics, and even the general "shape"
of the API, many things could go wrong - particularly around flow control and
errors. See, the use of `goto`s is emulated by c2rust using complicated
loop/match/break constructs, all of which went away completely, replaced by
Rust's "try" operator, `?`. Even trivial changes are an opportunity for bugs to
sneak in, and even moreso when there are hundreds.

Even though it is much easier to write correct code in safe Rust than in C,
logic bugs are obviously just as possible, particularly when performing very
large scale refactoring.

## Performance

In order to compare the performance of unsafe-libyaml and the safe Rust port, I
found a relatively large YAML document (~700 KiB) and built a very simple
benchmark using [Criterion].

In short:

|  | unsafe-libyaml | libyaml-safer |
|--|-------|------|
| Parse 700 KiB document | 38.258 ms |  37.447 ms |
| Emit 700 KiB document  | 18.491 ms | 17.549 ms |

_Hold for applause._

These benchmarks were run on an AMD Ryzen 9 3950X 16-Core CPU (3.5 GHz) running
Windows 11.

### Analysis

* **Important:** While I do consistently get slightly faster times for
  libyaml-safer, I don't consider this benchmark to be conclusive in any way.
* What these numbers _do_ indicate is that the port to safe, idiomatic Rust does
  not produce meaningfully slower code.
  * The justification for "unsafe" C and C++ libraries is often that some
    performance is left on the table with safe Rust, primarily due to bounds
    checking. On modern hardware this justification has always seemed a bit
    suspicious to me, and I think this is another piece of (circumstantial)
    evidence that it often just doesn't matter in practice.
  * It's certainly possible to build things where bounds checking makes a
    difference. This just isn't that.
* _If_ libyaml-safer is consistently faster for all workloads, here are some
  possible explanations:
  * C libyaml needs to assert on a couple of invariants (such as pointers not
    being NULL), and unsafe-libyaml performs these assertions even in release
    mode. Some of these invariants are statically checked in safe Rust, where
    `&T` can never be "NULL".
  * Due to the use of `std::io::BufRead` in the parser, the parser performs
    fewer in-memory copies.
* _Even if_ this benchmark is completely bogus, the performance of libyaml-safer
  is still well within tolerance for what is acceptable, considering the massive
  increase in maintainability and safety
* Zero effort was made to optimize things, other than just not choosing
  obviously slower safe alternatives.
  * One optimization that might make sense is to implement more efficient
    lookahead in the parser. Currently, there are many places where the scanner
    and parser perform many redundant bounds checks on the input buffer.

## Conclusion

This was a lot of fun.

I think, to me, and to a lot of software engineers who came from a C++
background, using Rust is not just about nebulous promises of safety and
correctness.

It's about bringing the joy back into programming.

We all like to design pretty, user-friendly APIs, we all like to produce things
that just feel high quality. Many of us don't get to do that, because the
realities of our industry often do not permit it. And so, does it spark joy?

Coding in Rust feels like having your cake and eating it too. When people are
productive in Rust, it is because it makes many things that used to be hard -
things that cause tears, blood, and burnout - are no longer as hard. The result
is not just safer and easier, it is also just as performant. So please - join
us.

#### Footnotes

- In this post I have preserved type names and function names from
  `unsafe-libyaml`, but as part of porting to _idiomatic_ Rust, I also did
  rename all types and enums following idiomatic Rust conventions.


[YAML]: https://yaml.org/
[serde_yaml]: https://crates.io/crates/serde_yaml
[libyaml]: https://pyyaml.org/wiki/LibYAML
[unsafe-libyaml]: https://github.com/dtolnay/unsafe-libyaml
[c2rust]: https://c2rust.com/
[`Result`]: https://doc.rust-lang.org/stable/std/result/enum.Result.html
[Criterion]: https://docs.rs/criterion/latest/criterion/
