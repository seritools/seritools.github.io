+++
title = "Rust9x update: Rust 1.76.0-beta"
+++

{{ image(src="/images/rust9x-logo.svg",
   alt="'Ferrissoft Visual Rust 9X', logo in the style of old Visual Studio logos (just a joke, not the real name of the project :) )") }}

20 months since the initial release, Rust9x is back, whether you like it or not!

I've spent the last couple of days migrating the changes from Rust 1.61-beta to Rust 1.76-beta, and
filling some of the holes in API support on the way.

See the project wiki on Github:
[`rust9x/rust`](https://github.com/rust9x/rust/wiki) and the [previous announcement
post](@/blog/2022-04-21-announcing-rust9x/index.md).

## What's new?

Other than the obvious update to Rust 1.76 with all its benefits, there are a few other changes:

### Backtrace support

Backtraces are now supported on at least Windows 98 and up when providing a supported `dbghelp.dll`
file, and are natively supported on Windows XP and up. It is still recommended to supply an updated
dbghelp.dll. See the [Limitations](https://github.com/rust9x/rust/wiki/Limitations) page for more
information.

### Thread parking support

Code that uses thread parking should now work! I've brought back the old generic implementation that
was removed from upstream before, as no *relevant* platform needs it anymore.

### x86_64 support?

I haven't tested it, but I've added a `x86_64-rust9x-windows-msvc` target, which should work on
XP 64-bit and up. Let me know whether it does :)

### (Update: 2023-12-30) Downloadable binary package

I've also figured out how to create an easily shareable binary package, so you can try it out
without bothering to build it yourself. See the installation instructions in the [project
wiki](https://github.com/rust9x/rust/wiki).

## Obligatory picture

{{ image(src="./pic.jpg", alt="Picture of the sample program running on a Windows 98 SE PC and aWindows XP laptop") }}
