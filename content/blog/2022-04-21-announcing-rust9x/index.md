+++
title = "Rust9x: Compile Rust code for Windows 95, NT and above"
+++

*Discuss on
[Reddit](https://www.reddit.com/r/rust/comments/u8skda/rust9x_compile_rust_code_for_windows_95_nt_and/),
[URLO](https://users.rust-lang.org/t/rust9x-compile-rust-code-for-windows-95-nt-and-above/74581),
[HN](https://news.ycombinator.com/item?id=31112273)*

*tl;dr:* see the project wiki on Github: [`rust9x/rust`](https://github.com/rust9x/rust/wiki)

You might remember my old post about the journey to [get at least a bit of Rust code working for
Windows 98 SE](@/blog/2020-05-26-compiling-rust-for-legacy-windows/index.md). Well, I'm back, this
time with a more ambitious package:

## Announcing Rust9x (Rust 1.61.0-beta)

{{ image(src="logo.svg", alt="'Ferrissoft Visual Rust 9X', logo in the style of old Visual Studio logos (just a joke, not the real name of the project :)") }}

> *Blazingly fast! Y2k compliant! Works everywhere!*

Over on [Github](https://github.com/rust9x/rust/) you'll find a forked Rust repository containing
two new targets, `i586-rust9x-windows-msvc` and `i686-rust9x-windows-msvc`, as well as all the
necessary changes to support most of what the Rust standard library offers, on *all\** 32-bit
Windows versions, including Windows 95, 98, Me, NT 3.51, NT 4.0, 2000, XP and Vista.

\* NT 3.1 and 3.5 were not tested (yet).

Please have a look at the [wiki](https://github.com/rust9x/rust/wiki) for information on how to set
up and use the toolchain if you so desire, and if you'd like to have a look under the hood, have a
look at the [commits](https://github.com/rust9x/rust/commits/rust9x) themselves, which have some
additional information about the changes, both in code and in the commit messages.

Do note that this is just a side project I've done for fun; I'll probably not keep it up to date
with upstream all the time.

If you have any questions, encounter issues (there surely are lots of
them left) or would like to join the project, feel free to contact me via the links in the footer,
and/or create an issue in the repositories. Thanks! <3

## Gallery

### Extremely simple file downloader using `ureq` and `clap`

This is a quick toy program I've whipped up to show some actual code/crates running. It uses
[`clap`](https://docs.rs/clap/latest/clap/) for command line parsing and `ureq` to do the request.

TLS is supported via `rustls` starting from Windows XP, as its dependency `ring` [needs
`RtlGenRandom`](https://github.com/briansmith/ring/blob/main/src/rand.rs#L272). I haven't looked
into it too much, but from what I could tell, this should be the only additional API dependency, so
by falling back to another way of getting randomness (probably compromising security in the process
:)) this should work all the way down to Windows 95/NT 4.0 as well.

With TLS support on Windows XP SP3 (real hardware):

{{ image(src="winxp_ureq_tls.jpg", alt="ureq and clap running with TLS enabled on Windows XP SP3") }}

Without TLS support on Windows 95 B (real hardware):

{{ image(src="w95b_ureq.png", alt="ureq and clap running on Windows 95 B") }}

### Sample program

Windows NT 3.51 (VM):

This one is missing the network tests as NT 3.51 and earlier never got WinSock 2 support.

{{ image(src="win351.png", alt="sample program running on Windows NT 3.51") }}

Windows 95 B (real hardware):

{{ image(src="w95b.png", alt="sample program running on Windows 95 B") }}

Windows XP (real hardware):

{{ image(src="winxp.jpg", alt="sample program running on Windows XP") }}

Windows 11 (real hardware):

{{ image(src="win11.jpg", alt="sample program running on Windows XP") }}
