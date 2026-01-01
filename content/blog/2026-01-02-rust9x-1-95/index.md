+++
title = "Rust9x update: Rust 1.93.0-beta"
+++

{{ image(src="/images/rust9x-logo.svg",
   alt="'Ferrissoft Visual Rust 9X', logo in the style of old Visual Studio logos") }}

You know the drill by now: Another year and 8-9 Rust versions later, it's time for another Rust9x
update!

Rust9x provides Windows downlevel support for the Rust standard library, bringing it to Windows 95,
98, ME, and most NT-based Windows versions. See the project wiki on Github:
[rust9x/rust](https://github.com/rust9x/rust/wiki).

## What's new?

- Rebased and partially reimplemented all fallback code ontop of **Rust 1.93-beta**.
- `target_vendor` is on the way to being deprecated. Rust9x now uses `target_family = "rust9x"`,
  instead.
- [`std::net::hostname`](https://doc.rust-lang.org/std/net/fn.hostname.html) has a fallback now
- Process stdio piping has gained a second fallback implementation, now that the upstream version
  only works on Vista and later.
- The compiler build configuration now sets `lld = true`, so `rust-lld.exe` is available. `rust-lld`
  doesn't prevent us from setting the subsystem and OS version to legacy values like modern
  `link.exe` versions do, so we can get rid of the post-build step that executed `editbin.exe` on
  built binaries. Check out the updated [rust9x-sample] for the new configuration.

[rust9x-sample]: https://github.com/rust9x/rust9x-sample

Feel free to hit me up on Discord if you'd like to help out or just chat about Rust9x!
