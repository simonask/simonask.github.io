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
simultaneously: It is both `&str`, `String`, and `std::str::Iter`, all in one.
Sometimes it is even the `std::str::Chars` iterator, which is used to iterate
over full UTF-8 codepoints within the string. The only way to determine which
abstraction it is currently emulating is to look at the context.

Thankfully - libyaml being a well-structured C project - there are plenty of
context clues. When the `STRING_DEL()` macro is being used, the caller hopefully
assumes ownership. When the `STRING_ASSIGN()` macro is being used, the string is
usually being borrowed and/or used as an iterator.

The interesting takeaway is this: For all of the different roles that
`yaml_string_t` takes on, it turns out there is an abstraction in the Rust
standard library that matches the use case precisely. I did not encounter a
single case in the libyaml source code where the use of strings did not map
directly to one of the _several_ many ways of dealing with strings in Rust.
Phew!

_Joyful aside:_ There is a similar type in libyaml, called `yaml_buffer_t`, and
it turned out to be almost very close to a `VecDeque` from the standard library.
Hooray!

### Error handling

Mercifully, C does not have exceptions. They are easily among the top 20
mistakes committed by humanity in the latter half of the 20th century, and don't
even come for me because I will die on this hill.

What this means is that the flow around error handling is surprisingly similar
between well-structured C code and idiomatic Rust code. Rust has the `?`
operator to make things easier, and one of the important affordances of
[unsafe-libyaml] (Rust) over [libyaml] (C) is that integer return codes are
replaced with a bespoke `Success` struct. Replacing this with [`Result`] turned
out to be [pretty
trivial](https://github.com/simonask/libyaml-safer/commit/dc639b92b5754f58e8e8f7979449ba6aca54c3a6).

Unfortunately, the story doesn't end here. Another thing that C lacks is "RAII",
which is just the thing where resources are automatically cleaned up when the
variables holding the resources go out of scope. In other words, the `Drop`
trait.

So what does an honest C programmer do when they have multiple failure modes,
but don't want to copy and paste their cleanup code between different branches,
all of which would be very prone to mistakes down the line?

That's right: `goto`.

Modern languages like Rust scoff at such perversions of structure and order,
while C stands in the corner with an evil smirk. There's just no good way around
it, and for all of its horrors, it is probably the least horrible option even in
modern C code.

Rust, having wisely chosen to not include this control flow primitive in its vocabulary,


## Making invalid states unrepresentable

## Freebies

## Performance

[YAML]: https://yaml.org/
[serde_yaml]: https://crates.io/crates/serde_yaml
[libyaml]: https://pyyaml.org/wiki/LibYAML
[unsafe-libyaml]: https://github.com/dtolnay/unsafe-libyaml
[c2rust]: https://c2rust.com/
[`Result`]: https://doc.rust-lang.org/stable/std/result/enum.Result.html
