+++
title = "Compiling Rust binaries for Windows 98 SE and more: a journey"
+++

*tl;dr:* It actually works, mostly! See the [conclusion](#conclusion) down below.

Did you know that [Rust](https://www.rust-lang.org/) has a [Tier 2 target](https://forge.rust-lang.org/release/platform-support.html) called `i586-pc-windows-msvc`? I didn't either, until a few days ago. This target disables SSE2 support and only emits instructions available on the original [Intel Pentium](https://en.wikipedia.org/wiki/P5_(microarchitecture)) from 1993.

So, for fun, I wanted to try compiling a binary that works on similarly old systems. My retro Windows of choice is Windows 98 Second Edition, so that is what I have settled for as the initial target for this project.

Some inspiration also came from [C# running on Windows 3.11](https://www.hanselman.com/blog/NETEverywhereApparentlyAlsoMeansWindows311AndDOS.aspx).

## Prerequisites

### Visual C++ Toolset

After a quick search online it seems that the Visual C++ 2005 toolset is the last one that officially supports building for Windows 98. A quick Windows 10 VM and Visual Studio installation later, I've copied the whole `Microsoft Visual Studio 8` folder over to my host machine, knowing from previous endeavours that the CLI tools in Microsoft's C/C++ toolset are pretty much portable, as long as the environment variables are set correctly. The toolset includes a handful of batch files like `vsvars32.bat`, setting the proper paths and variables automatically. I've copied it and altered all the paths to point to the correct locations on my host machine.

### Rust

Since I was sure that some tinkering with `rustc` and/or the standard library will be necessary, I've checked out the Rust repo. Some `scoop install python`, setting the correct host and target in the `config.toml`, and `python x.py build -i --stage 1 src/libstd` later, Rust and all the dependencies began compiling! After 26 Minutes and 18 Gigabytes the stage 1 compiler and standard library for `x86_64-pc-windows-msvc` and `i586-pc-windows-msvc` have been built!

Props to the people that wrote the extensive documentation inside of `config.toml.example` and [online](https://rustc-dev-guide.rust-lang.org/building/suggested.html), making the build not much harder than your regular ol' Rust crate!

In order to use the fresh new toolchain, we have to tell `rustup` about it:

```plain
rustup toolchain link win98 D:\RustProjs\rust\build\x86_64-pc-windows-msvc\stage1
```

## Let the ~~tinkering~~ linkering begin

First of all, I've created a new binary crate/project and changed the *Hello, world!* to *Hello, Windows 98!*, of course. To make sure that the project always uses our own toolchain, the default has to be overridden:

```plain
rustup override set win98
```

After confirming that the toolchain works by building with the default MSVC tooling (`cargo run --target i586-pc-windows-msvc`), I have tried building again, but this time from within the `vsvars32.bat` development environment:

```plain
cmd
call vsvars.bat
cargo clean
cargo build --target i586-pc-windows-msvc
```

But nope! For some reason Rust/Cargo tries to pass in the `x86_64-pc-windows-msvc` library object files, so `link.exe` rightfully errors with:

```plain
fatal error LNK1112: module machine type 'X86' conflicts with target machine type 'x64'
```

I think it has something to do with the full `vsvars32.bat` config, but I did not feel like debugging it further, so I have tried to use the old linker (and libs) only.

Calling `link.exe` without the `vsvars32.bat` environment does not output anything and just exits with an error code. In order to configure the MSVC environment just for the linker call, I have created a `linker.cmd` batch file:

```bat
@echo off
call vsvars32.bat
link.exe %*
```

To tell cargo to use another linker, we can add a `.cargo/config` file:

```toml
[build]
target = "i586-pc-windows-msvc"

[target.i586-pc-windows-msvc]
linker = 'D:\RustProjs\hello-w98\linker.cmd'
# show the linker command line
rustflags = ["-Z", "print-link-args"]
```

Now it looks like it has linked the files successfully, but there is no `.exe` file in sight! It took me an embarassingly large amount of time to get the idea to run the linking command manually:

```plain
error: linking with `D:\RustProjs\hello-w98\linker.cmd` failed: exit code: 1103
  ...
  = note: Setting environment for using Microsoft Visual Studio 2005 x86 tools.
          hello_w98.xw46fam95fk1t5e.rcgu.o : fatal error LNK1103: debugging information corrupt; recompile module
```

> [current-time seri here]
>
> It took me embarassingly long *again* when retracing the steps while writing this blog post, since I didn't bother checking what caused it to "fail successfully". Batch files don't work like rust expressions after all, so you have to append an `exit /B %ERRORLEVEL%` line to have the batch file exit with the exit code of the linker...

But who needs debugging information anyways? :)

```toml
[profile.dev]
debug = false

[profile.release]
debug = false
```

It has *finally* linked successfully, but the executable requires the VC 2005 redistributable, which I have installed on both the Windows 98 SE system and my host computer. The program has stayed stubborn:

{{ image(src="msvcr80.png", alt="hello-w98.exe - System Error: The code execution cannot proceed because MSVCR80.dll was not found. Reinstalling the program may fix this problem.") }}

Even putting the `MSVCR80.dll` that *comes with the tools I've linked the executable with* directly into the executable folder did not work! One more option to add to `rustflags`:

```toml
rustflags = ["-Z", "print-link-args", "-C", "target-feature=+crt-static", "-C", "link-args=unicows.lib"]
```

This is highly discouraged for regular applications, but so is trying to compile for Windows 98 targets I guess. :)

While I was at it, I have also added the [Microsoft Layer for Unicode](https://en.wikipedia.org/wiki/Microsoft_Layer_for_Unicode), also known as `unicows.lib/dll`. This library enables calling the "wide"/`W` variants of many Windows APIs, which I assumed would be necessary anyways, since Rust supports Unicode of course. Of course I didn't read quite enough on how to *actually* add the library to a program, but more on `unicows` later...

This executable actually runs fine on my host system now, and in the Windows 98 VM a more descriptive error message appears, one that means that the program is actually trying to run! The message informs us that the entry point `KERNEL32.DLL:AddVectoredExceptionHandler` could not be found. Looking this function up it seems it was added with Windows XP. It is time to get to work in the standard library!

{{ image(src="missing_add_vectored_98.png", alt="The HELLO-W98.EXE file is linked to missing export KERNEL32.DLL:AddVectoredExceptionHandler.") }}

## Intermission: KernelEx

[KernelEx](http://kernelex.sourceforge.net/) is a compatibility layer for Windows 9X/ME, allowing these systems to run some modern executables by providing missing system APIs. This works by patching the kernel in-memory via a device driver. I have tried running the executable with KernelEx throughout this journey, but `AddVectoredExceptionHandler` is not one of the APIs it provides.

## Browsing through the Rust source

### Creating a new target

I have actually done this part directly after setting up the Rust compilation by just searching for `i586-pc-windows-msvc`. In [`i586_pc_windows_msvc.rs`](https://github.com/rust-lang/rust/blob/1.43.1/src/librustc_target/spec/i586_pc_windows_msvc.rs) the target is defined by overwriting a few options of the regular `i686-pc-windows-msvc` target:

```rs
use crate::spec::TargetResult;

pub fn target() -> TargetResult {
    let mut base = super::i686_pc_windows_msvc::target()?;
    base.options.cpu = "pentium".to_string();
    base.llvm_target = "i586-pc-windows-msvc".to_string();
    Ok(base)
}
```

Especially of note are the [`TargetOptions`](https://github.com/rust-lang/rust/blob/1.43.1/src/librustc_target/spec/mod.rs#L560-L814), which define lots and lots of interesting target-specific values, like `crt_static_default` for example, rendering `target-feature=+crt-static` redundant.

The actual mapping of the target name/triplet to the spec file happens [here](https://github.com/rust-lang/rust/blob/1.43.1/src/librustc_target/spec/mod.rs#L335). After copying the spec file as `i586-pc-windows-msvclegacy.rs`, adding it to the target list and changing the `target` entry in the `config.toml` file, the new target is up and ready to be used!

I have *also* added `unicows.lib` to libstd's [`build.rs`](https://github.com/rust-lang/rust/blob/1.43.1/src/libstd/build.rs#L45) file for good measure (still without reading about it):

```rs
// ...
} else if target.contains("windows") {
    if (target == "i586-pc-windows-msvclegacy") {
      println!("cargo:rustc-link-lib=unicows");
    }
    println!("cargo:rustc-link-lib=advapi32");
    println!("cargo:rustc-link-lib=ws2_32");
    println!("cargo:rustc-link-lib=userenv");
} else if target.contains("fuchsia") {
// ...
```

### Yanking out `AddVectoredExceptionHandler`

The [`AddVectoredExceptionHandler`](https://docs.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-addvectoredexceptionhandler) Windows API function is used by Rust to provide a nicer error experience in case of a [stack overflow](https://github.com/rust-lang/rust/blob/1.43.1/src/libstd/sys/windows/stack_overflow.rs). However, as seen in other targets, this handling is completely optional. Incidentally the [UWP implementation](https://github.com/rust-lang/rust/blob/1.43.1/src/libstd/sys/windows/stack_overflow_uwp.rs) is one of these targets, so we can just replace the regular implementation with that one (and even keep the `Handler` type, so no further source changes are needed).

I wanted to not affect the host Windows target, however, so I have created a copy of the whole `libstd/sys/windows` folder as `libstd/sys/windows_legacy`, adding an admittedly wonky compilation condition to `libstd/sys/mod.rs`:

```rs
// ...
} else if #[cfg(all(windows, not(target_arch = "x86")))] {
    mod windows;
    pub use self::windows::*;
} else if #[cfg(all(windows, target_arch = "x86"))] {
    mod windows_legacy;
    pub use self::windows_legacy::*;
} else if #[cfg(target_os = "cloudabi")] {
// ...
```

This changes the `sys` module used for any 32-bit Windows of course, instead of just `i586-pc-windows-msvclegacy`, but I couldn't find any way to conditionally compile for a specific target triple, since that is not directly exposed as a configuration option. Since my host target is 64-bit, that was good enough™, though. :)

All C FFI declarations used for interfacing with Windows are neatly packed in [`c.rs`](https://github.com/rust-lang/rust/blob/1.43.1/src/libstd/sys/windows/c.rs), where I removed the declarations for `AddVectoredExceptionHandler`, `SetThreadStackGuarantee` and a few structs needed for those, since the compilation settings in the Rust codebase rightfully deny any unused code.

In a "proper" implementation I would maybe expose another `target_something` configuration option to match against, potentially even down to compatibility info per API per version.

After this change and another recompile of the sample application the `AddVectoredExceptionHandler` error message is gone, only to be replaced by a message about a missing `KERNEL32.DLL:RtlCaptureContext` export:

{{ image(src="missing_rtl_98.png", alt="The HELLO-W98.EXE file is linked to missing export KERNEL32.DLL:RtlCaptureContext.") }}

I've quickly googled to find out whether KernelEx supports this function, and it does! Turning KernelEx on, opening the executable again, and I am greeted by my very first Rust output on Windows 98!

{{ image(src="kernelex_works.png", alt="The working program and KernelEx settings") }}

Granted, using KernelEx is cheating, but it gave me the confidence that a true legacy Windows compatible executable is within the realms of possibility.

## The hunt for `RtlCaptureContext`, part one

As with `AddVectoredExceptionHandler`, [`RtlCaptureContext`](https://docs.microsoft.com/en-us/windows/win32/api/winnt/nf-winnt-rtlcapturecontext) was introduced with Windows XP, so we will have to get rid of it. I couldn't find a reference to this function in the Rust source (more on that in part two), so let's get out some reversing tools!

Since a [Portable Executable (PE)](https://en.wikipedia.org/wiki/Portable_Executable) file usually has an `.idata` section with all the import information, I wanted to validate it with [Dependency Walker](https://www.dependencywalker.com/). That tool froze when loading the test executable for some reason, so I switched to the open-source alternative called [Dependencies](https://github.com/lucasg/Dependencies):

{{ image(src="dependencies_gui.png", alt="DependenciesGUI with hello-w98.exe open") }}

So yes, the import is definitely there. The next tool I have utilized was [Ghidra](https://ghidra-sre.org/), a reverse engineering and code analysis toolset. Ghidra can auto-analyze the binary to find all usages of the imported functions, for example. But, to my surprise, the import is not used at all according to the auto-analysis! So the *most sensible* thing to do is to open a hex editor, find the string `RtlCaptureContext` and replacing it with an import name that definitely exists in Windows 98, filling any additional space with `\0`. I have chosen `GetCurrentThread` just because it happened to be the import prior to `RtlCaptureContext`.

The program runs to completion without any errors now, yay! But it also doesn't output anything, oops. Even when putting `unicows.dll` next the executable, it just doesn't output anything. At that point I've tried changing the `Hello, Windows 98!` implementation to one with byte strings, writing directly to `stdout`:

```rs
use std::io::Write;

fn main() {
    let stdout = std::io::stdout();
    let mut lock = stdout.lock();
    lock.write(b"Hello, Windows 98!").unwrap();
}
```

... which did not change a thing.

> [current-time seri here]
>
> Yes, I know, anyone with some form of knowledge about the Windows console knows that this was an exercise in futility, as I remember/find out below :)

## OllyDbg to the rescue

Let's summarize:

- We are on Windows 98 SE
- without debugging info/symbols
- in a release-compiled executable that doesn't output anything

How are we going to debug the problem? We use [OllyDbg 1.10](http://www.ollydbg.de/), which runs perfectly fine on Windows 98, like in the good ol' times!

After firing up Olly and loading the executable, checking that everything works as expected, I've gone back to the import list to find out which kernel function would be a good candidate to set a breakpoint on. `WriteFile` seemed like a good candidate, so I've reloaded the executable in Olly, right-clicked in the disassembly window, selected *Search for → all intermodular calls*, found two call to `WriteFile`, set breakpoints on both, pressed "play" to have the program running until a breakpoint and ...

{{ image(src="olly_writefile.png", alt="OllyDbg with WriteFile breakpoints set") }}

... it's not called.

This means that `WriteFile` probably wasn't the correct function, or it didn't even try calling it because of *something*. I've went with the first option first and looked again for other suspicious function imports. None of them seemed related to writing to the console though, so I have tried another way: finding the string the program writes to the console itself!

Next to the *all intermodular calls* context menu entry there is an *all referenced strings* entry. That search *did* yield the `Hello, Windows 98!` string, but it seems that OllyDbg stuggled with the static analysis, and marked the whole area that uses the string as data. Even after helping Olly by explicitly marking it as data and re-analyzing the binary, no additional calls seemed to be found.

I've switched back to Ghidra, where the correct function was obvious: `WriteConsoleW`. Noting the addresses where it is used, I've gone back to OllyDbg to see what it shows at those locations. And there it is, a normal call to `WriteConsoleW`. How did it not show in the list of intermodular calls? Well...

{{ image(src="olly_writeconsole.png", alt="OllyDbg showing the wrong function name") }}

I've sorted the list by the "Destination" column, which shows a wrong function name! Back on track, I've set breakpoints on both of these calls, pressed "go", and:

{{ image(src="olly_writeconsole_bp_before.png", alt="OllyDbg hitting one of the WriteConsoleW breakpoints") }}

The breakpoint hits, we see our string as a parameter on the stack, and step over the call. OllyDbg has a nice status window that shows the current register values, and also other values helpful for debugging Windows applications, like the [`GetLastError`](https://docs.microsoft.com/en-us/windows/win32/api/errhandlingapi/nf-errhandlingapi-getlasterror) value:

{{ image(src="olly_writeconsole_bp_after.png", alt="OllyDbg showing the last error code after calling WriteConsoleW") }}

`ERROR_CALL_NOT_IMPLEMENTED (0x00000078)` ... makes sense on Windows 98! At that point I had realized:

- that the Windows NT console Rust targets is Unicode/UTF-16, so my attempt at writing a byte string directly to `stdout` was converted to unicode anyways (notice how Olly also shows the string passed in as parameter as `UNICODE`), and
- that I have added `unicows` incorrectly, since `WriteConsoleW` definitely is one of the [supported functions](https://web.archive.org/web/20041118165629/http://msdn.microsoft.com/library/en-us/mslu/winprog/mslu_w_apis.asp).

## Get `unicows` working

Since the error undoubtedly points to `unicows` not working correctly, I've finally taken a closer look at the documentation [[1]](https://web.archive.org/web/20051208110500/http://msdn.microsoft.com/library/en-us/mslu/winprog/microsoft_layer_for_unicode_on_windows_95_98_me_systems.asp) [[2]](https://web.archive.org/web/20050220043829/http://msdn.microsoft.com/library/en-us/mslu/winprog/compiling_your_application_with_the_microsoft_layer_for_unicode.asp) and I've found a nice blog post explaining [how it works](http://archives.miloush.net/michkap/archive/2005/05/01/413835.html) under the hood.

I have removed the old `unicows.lib` include from both the standard library's `build.ts` and changed the project's `rustflags` to include basically exactly what the documentation says:

*formatted for readability, actual TOML arrays have to be inline*

```toml
[build]
target = "i586-pc-windows-msvclegacy"

[target.i586-pc-windows-msvclegacy]
linker = 'D:\RustProjs\hello-w98\linker.cmd'
rustflags = [
  "-Z", "print-link-args",
  "-C", "target-feature=+crt-static",
  "-C", 'link-args=/nod:kernel32.lib
    /nod:advapi32.lib
    /nod:user32.lib
    /nod:gdi32.lib
    /nod:shell32.lib
    /nod:comdlg32.lib
    /nod:version.lib
    /nod:mpr.lib
    /nod:rasapi32.lib
    /nod:winmm.lib
    /nod:winspool.lib
    /nod:vfw32.lib
    /nod:secur32.lib
    /nod:oleacc.lib
    /nod:oledlg.lib
    /nod:sensapi.lib
         unicows.lib
         kernel32.lib
         advapi32.lib
         user32.lib
         gdi32.lib
         shell32.lib
         comdlg32.lib
         version.lib
         mpr.lib
         rasapi32.lib
         winmm.lib
         winspool.lib
         vfw32.lib
         secur32.lib
         oleacc.lib
         oledlg.lib
         sensapi.lib'
]
```

After patching out `RtlCaptureContext` once again, I was greeted with my first truly Windows 98 SE compatible executable! Sleeping over the results, I've realized that `unicows` still wasn't included correctly...

## Get `unicows` *properly* working

The next day I've followed my train of thought about the linking process. Looking at the final linker command line:

```plain
... "advapi32.lib" "ws2_32.lib" "userenv.lib" "libcmt.lib"
"/nod:kernel32.lib" "/nod:advapi32.lib" "/nod:user32.lib"
"/nod:gdi32.lib" "/nod:shell32.lib" "/nod:comdlg32.lib"
"/nod:version.lib" "/nod:mpr.lib" "/nod:rasapi32.lib"
"/nod:winmm.lib" "/nod:winspool.lib" "/nod:vfw32.lib"
"/nod:secur32.lib" "/nod:oleacc.lib" "/nod:oledlg.lib"
"/nod:sensapi.lib" "unicows.lib" "kernel32.lib" "advapi32.lib"
"user32.lib" "gdi32.lib" "shell32.lib" "comdlg32.lib"
"version.lib" "mpr.lib" "rasapi32.lib" "winmm.lib"
"winspool.lib" "vfw32.lib" "secur32.lib" "oleacc.lib"
"oledlg.lib" "sensapi.lib"
```

We can see that `"advapi32.lib"`, `"ws2_32.lib"`, `"userenv.lib"`, and `"libcmt.lib"` are added before the linker args from the `.cargo/config`, meaning the respective `/nod` (`/NODEFAULT`; disable default import) entries are without effect, and functions imported from these would not be wrapped by `unicows`, as the linker priority order goes from left to right. So where do these imports come from? I have remembered seeing them in the [`build.rs`](https://github.com/rust-lang/rust/blob/1.43.1/src/libstd/build.rs#L45-L47) of the standard library. Now to find out how they get there.

> [current seri here]
>
> Starting from here, all the Rust source code links will point to the tree at the exact commit I was at when working on this, since some of the features (e.g. [#70093](https://github.com/rust-lang/rust/issues/70093)) are newer than the 1.43.1 that is current at the time of writing.

After a bit of searching I have found `librustc_codegen_ssa/back/link.rs`, with [`linker_with_args`](https://github.com/rust-lang/rust/blob/bad3bf622bded50a97c0a54e29350eada2a3a169/src/librustc_codegen_ssa/back/link.rs#L1405) looking promising! In fact, the called [`link_local_crate_native_libs_and_dependent_crate_libs`](https://github.com/rust-lang/rust/blob/bad3bf622bded50a97c0a54e29350eada2a3a169/src/librustc_codegen_ssa/back/link.rs#L1262) and [`add_upstream_native_libraries`](https://github.com/rust-lang/rust/blob/bad3bf622bded50a97c0a54e29350eada2a3a169/src/librustc_codegen_ssa/back/link.rs#L1903) look even more promising. To make sure this is the place that actually adds the libraries to the linker command line, I have just added a `println!` in the `for` loop at line 1935:

```rs
println!("lib: {}: {:?}", name, lib.kind);
```

Seems to be the right place! The call to `add_upstream_native_libraries` is actually wrapped in a condition checking for a compiler debugging (`-Z`) option, allowing us to deactivate the addition of these libraries:

```rs
if sess.opts.debugging_opts.link_native_libraries {
    add_upstream_native_libraries(cmd, sess, codegen_results, crate_type);
}
```

This means adding `-Z link_native_libraries=no` to the ever-growing list of `rustflags` should do the trick. Additionally, I have added the libraries required by `libstd` to the end of the linker args, after `unicows`:

```toml
rustflags = [
  "-Z", "print-link-args",
  "-Z", "link_native_libraries=no",
  "-C", "target-feature=+crt-static",
  "-C", 'link-args=/nod:kernel32.lib
  /nod:advapi32.lib
  /nod:user32.lib
  /nod:gdi32.lib
  /nod:shell32.lib
  /nod:comdlg32.lib
  /nod:version.lib
  /nod:mpr.lib
  /nod:rasapi32.lib
  /nod:winmm.lib
  /nod:winspool.lib
  /nod:vfw32.lib
  /nod:secur32.lib
  /nod:oleacc.lib
  /nod:oledlg.lib
  /nod:sensapi.lib
       unicows.lib
       kernel32.lib
       advapi32.lib
       user32.lib
       gdi32.lib
       shell32.lib
       comdlg32.lib
       version.lib
       mpr.lib
       rasapi32.lib
       winmm.lib
       winspool.lib
       vfw32.lib
       secur32.lib
       oleacc.lib
       oledlg.lib
       sensapi.lib
       ws2_32.lib
       userenv.lib
       libcmt.lib'
]
```

Now the linker command line looks correct and all wrapped libraries should be properly wrapped by `unicows`! `link_native_libraries=no` is not a perfect solution, but good enough for now.

## The hunt for `RtlCaptureContext`, part two

Until this point, each and every new executable needs to be edited to "remove" the `RtlCaptureContext` import that is still added somehow.

In order to find the source, the tool `dumpbin`, part of the MSVC toolset, comes to help. First I've confirmed that `RtlCaptureContext` is actually imported:

```plain
dumpin /imports hello-w98.exe
```

Then I've gone through all libraries listed in the linker call, in order, to find the one that mentions `RtlCaptureContext` somewhere in its defined symbols. Of course `libstd` is the one:

```plain
dumpbin /symbols "D:\RustProjs\rust\build\x86_64-pc-windows-msvc\stage1\lib\rustlib\i586-pc-windows-msvclegacy\lib\libstd-ca1f5c4034a86a91.rlib" | rg Rtl
239 00000000 UNDEF  notype       External     | _RtlCaptureContext@4
```

Let's open the file in Ghidra, which detects it as having 32 subfiles (first time I've seen such a thing, interesting!):

{{ image(src="ghidra_libstd.png", alt="Ghidra project listing, showing all 32 added object files inside the rlib", link=false) }}

There is probably a better way to do this, but I have opened them one by one until I found the one importing `RtlCaptureContext`. In the end it was the one starting with `5pja0`, and the function calling our target was `backtrace::backtrace::trace_unsynchronized`. Doh, `libstd` of course has dependencies that are not part of the rustc tree! And [*there*](https://github.com/rust-lang/backtrace-rs/blob/0d859d82d782a6784734cc618193907a354add46/src/backtrace/dbghelp.rs#L88) is the call, finally.

Searching for `backtrace` in the project reveals that there is a compilation option in `config.toml` that allows turning off backtraces:

```toml
[rust]
# Whether or not `panic!`s generate backtraces (RUST_BACKTRACE)
backtrace = false
```

So I have turned backtraces off, recompiled everything, checked the imports and `RtlCaptureContext` 🦀 is 🦀 gone 🦀! The executable seems to be fully compatible now, without modifications, with Windows 98 SE, at least with this *very* limited subset of features needed for displaying `Hello, Windows 98!`.

*Sidenote:* Disabling backtrace support reduced the binary size by about 50KiB, interesting!

## Windows NT 3.51 and debug symbol support

Since the executable now works on Windows 98 SE, I've tried Windows NT 3.51 next. This worked out of the box (and should be easier anyways since it already supports the `W` unicode APIs out of the box). It works without further changes because NT 3.51 uses the [same PE subsystem version `4`](http://www.malsmith.net/blog/nt351-runs-40-applications/) that NT4 and Windows 98 SE use, both of which are ([more or less](http://www.malsmith.net/blog/visual-c-visual-history/)) supported by the VC2005 linker.

{{ image(src="nt351.png", alt="'Hello Windows 98!' on Windows NT 3.51") }}

It seems that to go even lower, to NT 3.1, a subsystem version of `3.10` is needed. The [original way](http://www.malsmith.net/blog/pe-subsystem-2/) of setting that was to use `verfix.exe` from the NT 3.1 SDK, and looking for modern alternatives I have found that `editbin.exe` of the VC toolset has the same functionality as well. Just watch out for [behavior changes](http://www.malsmith.net/blog/pe-subsystem-version/) depending on the subsystem version.

But wait, the subsystem can be set via the linker options directly... Can the modern VC2019 linker be used? I have added `/SUBSYSTEM:CONSOLE,3.10` to the linker args and tried again: `link.exe` just gives a warning, stating that the subsystem version is invalid and will be replaced by the default version (`6.0`; Vista or later). It seems that the linker only allows supported subsystem versions.

What about `editbin.exe`? `editbin /SUBSYSTEM:CONSOLE,3.10 hello-w98.exe` gives the same warning but changes the version regardless! Linking with the modern linker against modern libraries causes it to import unavailable APIs again (probably from the modern MSVCRT), so the way to go is to still load the old `vsvars32.bat` environment, but call the modern linker via its full path. And, by using the modern linker, there are now no problems loading the debugging information anymore either!

## Conclusion

With relatively small changes to the standard library and compilation settings, and by using old linker libraries, we can make simple applications work on legacy Windows versions already. It's debatable of how much use all of this is, but the journey was fun nonetheless!

If someone is interested in bringing this further, here are some of the open TODOs and ideas I've gathered while working on this:

- Go through [`src\libstd\sys\windows\c.rs`](https://github.com/rust-lang/rust/blob/bad3bf622bded50a97c0a54e29350eada2a3a169/src/libstd/sys/windows/c.rs) and check/rewrite ALL the APIs in a legacy-compatible way
  - I've thought about maybe adding additional [configuration options](https://doc.rust-lang.org/reference/conditional-compilation.html#set-configuration-options), one for each Windows version or API set, allowing the standard library to target any API level specifically. This could be done [here](https://github.com/rust-lang/rust/blob/bad3bf622bded50a97c0a54e29350eada2a3a169/src/librustc_session/config.rs#L672), for example.
  - Note that the current Microsoft docs on the Windows API do not contain information about API support for versions older than Windows 2000. For that you will need an old MSDN Library as reference.
- Implement an ANSI/Codepage based version of `OsStr`/`OsString` for Windows 9x systems, removing the need for `unicows`
- Minimize configuration needed (esp. when using `unicows`)
  - Ideally `unicows` would not be needed anyways, so all of the tricks of disabling the addition of native libs could be skipped
  - If `unicows` should stay supported (or toggleable), add it to the linker args generation logic in rustc
- Get `backtrace`s working again, if possible
- Get dynamic MSVCRT linking working
- Try out and test other VC toolsets and Windows versions to see how low you can go—it doesn't seem like anything should prevent going down to Windows 95. [Win32s](https://en.wikipedia.org/wiki/Win32s) seems unlikely though.

You can find the `rustc` changes I've made over on GitHub in the [`windows-legacy-poc` branch](https://github.com/seritools/rust/tree/windows-legacy-poc), if you're at all interested! My `config.toml` basically looks like this:

```toml
[build]
build = "x86_64-pc-windows-msvc"
target = ["i586-pc-windows-msvclegacy"]

[rust]
backtrace = false
```

Also see the blog's footer for my contact information if you have any comments, questions or suggestions.

Thank you for reading!

Discussion on [/r/rust](https://www.reddit.com/r/rust/comments/gr0xqa/compiling_rust_binaries_for_windows_98_se_and/), [HN](https://news.ycombinator.com/item?id=23313577), [Rust user forum](https://users.rust-lang.org/t/compiling-rust-binaries-for-windows-98-se-and-more-a-journey/43283)