+++
title = "Rust9x update: Rust 1.84.0-beta"
+++

{{ image(src="/images/rust9x-logo.svg",
   alt="'Ferrissoft Visual Rust 9X', logo in the style of old Visual Studio logos") }}

Another year has passed, oh how the time flies! Time for some Tier 4 target action once again!

Rust9x adds support for Windows 9x and older NT-based Windows versions to Rust. See the project wiki
on Github: [`rust9x/rust`](https://github.com/rust9x/rust/wiki).

## What's new?

- Rebased and reimplemented all fallback code ontop of the ~29600 commits of progress from Rust 1.76
  to Rust 1.84-beta
- The thread parker now only conditionally falls back to the basic, generic one. If supported by the
  OS, Rust9x will instead choose the `NtCreateEvent`-based parker or the
  `futex`/`WaitOnAddress`-based one.
- Better `RwLock` implementation: Rust9x now uses the upstream `queue` implementation instead of
  falling back to a simple `Mutex`. Similarly, `Once` also uses the `queue` implementation now.
  - Since these fallbacks rely on the thread parker only, they should run much faster with the new
    thread parker fallbacks above, too!
- Cleaned up the fallback implementations to intrude less on normal non-rust9x Rust `windows-msvc`
  targets.

## What's next?

- I'll try to keep up with 1.84 beta branch changes for now.
- TLS is as bad as ever. Even with the native TLS callback, `Drop` impls will not run on 9x. I still
  hope to get to the bottom of this!
- Windows NT 3.51 is currently broken (failing on `_chkstk` stack probes when doing stdio, but
  _only_ then). NT 4 and Windows 95 RTM work fine.
- Reimplementing the [`fileextd.lib` fallbacks](https://github.com/rust9x/rust/issues/43) in Rust
  will give us a much more complete `std::fs` implementation on Windows XP (and possibly earlier
  NT-based systems).
- The [fallback](https://github.com/rust9x/rust/issues/5) from WinSock2 to WinSock 1.1 should
  actually be pretty simple to implement. It would give us networking support without extra DLL
  dependencies on Windows 95 RTM and NT 3.51.

For now I won't publish a precompiled package (the `install` scripts are insanely slow on Windows).
I'd rather work on the above fixes and leave compiling as an exercise to the reader :\^). You can
find build instructions in the [Wiki](https://github.com/rust9x/rust/wiki).

Please don't expect too much, though - [Path of Exile 2](https://pathofexile2.com/early-access) is
launching into EA soon, too, so I'll be busy with that!

Feel free to hit me up on Discord if you'd like to help out or just chat about Rust9x!
