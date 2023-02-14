---
title: "Linking Rust and C: Practicing Symbol Hygiene"
date: 2023-02-14T16:08:02.796Z
---

{{<html>}}
<style>
code { white-space: nowrap; }
</style>
{{</html>}}

This is a post about the bits and pieces I learned while trying to fix a symbol
leakage problem for YJIT-enabled Ruby. The symbols I'm talking about are the
labels in object, executable, and shared library files that help linkers work.

Thinking back to my first few contacts with linkers, I found them somewhat
mysterious. I think this is because I get errors from compilers more frequently
than from linkers. Also, compared to compilers, linkers are generally in a
worse vantage point for giving context-rich error messages that use familiar
programming language level terms.

This post is pretty much all about linking on Unix-like systems. I hope my
adventures with various linkers here can cut their mystic a bit.

## Linking two Rust `staticlibs` together?

A good reason for caring about the list of symbols available in a static
library is that it helps to avoid link-time conflicts. Generally speaking,
linkers complain when there are multiple symbol definitions with identical
names. There is ambiguity as to which definition the linker should pick in
these situations, so they choose to error out. Savvy users might know about the
`--allow-multiple-definition` option or equivalents, so it's not an
insurmountable problem. However, these options are instructions for resolving
conflicts, each with different tradeoffs. It's nicer for static library
consumers when there are no conflicts in the first place.

This brings us to [`rust-lang/rust#44322`]. To summarize the issue, two Rust
staticlibs can link without conflicts when [LTO] is off and fail to link when
LTO is on. As of Rust 1.67.1, support for generating staticlibs that link well
together is spotty.

## Impact to YJIT

[CRuby] can be configured to build as a static library or a shared library.
YJIT, the Rust component, is optional. Users can choose to build Ruby without
it.

Let's say you've been linking against the CRuby static library before YJIT was
an option. You also link against some unrelated Rust staticlibs, maybe prebuilt
ones. A new CRuby release rolls around, and you decide to try out YJIT. You
build the new YJIT-enabled CRuby static library and pass it to your old build
setup. What happens?


>lib2.cgu-0.rs:(.text.rust_eh_personality+0x0): multiple definition of `rust_eh_personality'
>...
>
>collect2: error: ld returned 1 exit status


Oof. Not fun. Who's `collect2` and what's `rust_eh_personality` anyways? Maybe
you decide to disable YJIT because it's the new thing that causes build
problems for you. YJIT loses a user.

Zooming out from this admittedly contrived user story, the general point here
is that YJIT, by changing the list of symbols that are visible in Ruby's static
library, can cause build failures for consumers. The list of symbols available
in a static library has a close relation to its public interface, so you
shouldn't even need to fully subscribe to [Hyrum's Law] to see how changing the
list can cause problems.

## How linkers handle static libraries (archive libraries) {#earlier}

Structurally, static libraries are not much more than a collection (an archive,
if you will) of object files. Passing a static library to the linker is not
equivalent to passing all of its member objects, though. Traditionally,
linkers pull objects out of archives on an as-needed basis. Say we have an
archive, `libmy.a`. Inside, we have `a.o`, which defines `_a`, and
`b.o` which defines `_b`. When we only need `_a`, the linker would only
link against `a.o`; `b.o` would not be extracted from the archive. Some
static libraries take advantage of this behavior to help users get smaller
final images. For example, in the musl libc archive Alpine Linux ships,
each library function gets their own object (try `ar t /usr/lib/libc.a`).
Only the libc functions you use end up in the final image.

*This selective linking behavior can mask multiple definition errors.* Let's
look at a concrete reproducer for [`rust-lang/rust#44322`] and look at what
objects the linker chooses to extract.

```sh
#!/bin/sh
#
# This is repro.sh. It assumes you're working with GNU Binutils.
# Run in an empty directory.
CC=cc
RUSTC='rustc +1.67.1' # Your distro's rustc might not support this +... syntax.
CODEGEN_OPTS='-O'
if [ "$1" = 'lto' ]; then
    CODEGEN_OPTS="${CODEGEN_OPTS} -Clto"
fi

set -o errexit
set -o nounset
set -o xtrace

# Make libone.a. It defines the "_one" symbol.
printf '%s' '#[no_mangle] pub extern "C" fn one() -> std::ffi::c_int { 1 }' \
    | $RUSTC $CODEGEN_OPTS --crate-type=staticlib -o libone.a -

# Make libtwo.a. It defines the "_two" symbol.
printf '%s' '#[no_mangle] pub extern "C" fn two() -> std::ffi::c_int { 2 }' \
    | $RUSTC $CODEGEN_OPTS --crate-type=staticlib -o libtwo.a -

# Make my.o, which calls both one() and two()
printf '%s' 'int one(void); int two(void); void my(void) { one(); two(); }' \
    | $CC -x c -std=c99 -c -o my.o -

# Link everything together to produce libmy.so.
# `--trace-symbol=...` is a GNU linker feature.
$CC -shared -Wl,--trace-symbol=one -Wl,--trace-symbol=two -Wl,--trace-symbol=rust_eh_personality -o libmy.so my.o libone.a libtwo.a

: 'Success!'
```

Run `sh repro.sh lto` to see the link error:

```shell
$                 # Some filtering for brevity
$ sh repro.sh lto 2>&1 | cut -d ' ' -f2-
<snip>
my.o: reference to one
my.o: reference to two
libone.a(libone.rust_out.7f80e3a4-cgu.0.rcgu.o): definition of one
libone.a(libone.rust_out.7f80e3a4-cgu.0.rcgu.o): definition of rust_eh_personality
libtwo.a(libtwo.rust_out.7f80e3a4-cgu.0.rcgu.o): definition of two
libtwo.a(libtwo.rust_out.7f80e3a4-cgu.0.rcgu.o): in function `rust_eh_personality':
multiple definition of `rust_eh_personality'; libone.a(libone.rust_out.7f80e3a4-cgu.0.rcgu.o):/rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/personality/gcc.rs:244: first defined here
error: ld returned 1 exit status
```

The linker's tracing output shows the process that leads to the conflict.
Here is an abridged version of the story to highlight the important pieces:

```text
# expected, due to what we wrote in `my()`
my.o: reference to one
my.o: reference to two

# The linker extracts `libone.rust_out.o` to satisfy the reference to `one`.
# The same object defines `rust_eh_personality`
libone.a(libone.rust_out.o): definition of one
libone.a(libone.rust_out.o): definition of rust_eh_personality

# The linker extracts `libtwo.rust_out.o` to satisfy the reference to `two`
libtwo.a(libtwo.rust_out.o): definition of two

# `libtwo.rust_out.o` also defines `rust_eh_personality`, which is already
# defined by `libone.rust_out.o` from earlier! Multiple definitions.
libtwo.a(libtwo.rust_out.o): multiple definition of `rust_eh_personality'; libone.a(libone.rust_out.o): first defined here
```

Ok, but then why does it work when we disable LTO? Let's see if the linker
trace explains it:

```shell
$ sh repro.sh 2>&1 | cut -d ' ' -f2-
<snip>
my.o: reference to one
my.o: reference to two
libone.a(libone.rust_out.7f80e3a4-cgu.0.rcgu.o): definition of one
libtwo.a(libtwo.rust_out.7f80e3a4-cgu.0.rcgu.o): definition of two
: 'Success!'
```

No extracted objects define `rust_eh_personality`, so there is no conflict!

Under LTO, because of the way Rust chooses to split the staticlib into objects,
`rust_eh_personality` comes along for the ride when linking against `one()`
and/or `two()`. This is a *leak of implementation detail*. It's reasonable to
expect `one` and only `one` to be available in `libone.a`, because that's all
we wrote in the source, but that's not what we get. `rust_eh_personality` is
part of the Rust's standard library ("stdlib" here on out). In fact, `libone.a`
defines many more symbols:

```shell
$ nm --just-symbols --defined-only --extern-only libone.a | wc -l
846
```

That's a large interface for a one-line library!

## `ld -r`, also known as "partial linking"

To avoid potential conflicts, our goal is to trim the output of the `nm`
invocation above. Ideally, it would just have a single line that says `one`.

In our simple example, `one()` and `two()` don't use Rust standard library
functions, so the objects that define them in the archive don't need to refer
to other archive member objects. When we start using standard library functions
like YJIT, we have a bit of a dilemma. With ELF and Mach-O, as far as I know,
there is no way make a symbol available solely to other members of the archive;
if it's defined and "extern" (more about this later), it's available to objects
inside and outside of the archive.[^8] To satisfy our stdlib functions usages,
Rust can choose to put precompiled objects in the archive, but in doing so,
those symbols also become part of the external interface of the staticlib!

If you believe this limitation of the object code format, you can see why using
`strip(1)` on the archive directly doesn't quite work. If we strip away symbols
that we use but do not intend to make available externally, internal references
within the archive break.

So to trim the list of available symbols, having multiple objects in the
archive is a bit of a problem. It'd be nice if we can deal with just one
object file. That wouldn't directly solve the problem, but it'd help. Luckily, there
is a kind of program that combines multiple object files into one file --- the
linker!

It's common to use the linker to combine multiple object files into a shared
library or an executable. A lesser-known operation is merging multiple object
files into one object file. It's not a feature that all the linkers have, but
it's supported by the GNU linker, Xcode's linker (ld64), and [illumos'
linker](https://github.com/illumos/illumos-gate/blob/8b0ccc49c4d0c1ac61668d047d9e8f14ca9ff6f3/usr/src/man/man1/ld.1#L653-L663).
Linux uses this functionality to build kernel modules; the Glasgow Haskell Compiler
apparently also [uses it](https://github.com/rui314/mold/issues/289#issue-1110017351).
On Linux platforms that use [GNU Binutils](https://www.gnu.org/software/binutils/), it
should work pretty well.

The macOS manual page for `strip(1)` actually
[mentions](https://github.com/opensource-apple/cctools/blob/fdb4825f303fd5c0751be524babd32958181b3ed/man/strip.1#L117-L129)
using `ld -r` and `strip` to trim the symbol list down to an intended list ---
`ld -r` seems to be the right path!

CRuby already happens to have a configuration that uses `ld -r`. It's for the
original flavor of DTrace first shipped in Solaris, nowadays more easily
accessible through [illumos]. There is a [link][ruby-dtrace-marc] in the
Makefile to a 2005 [mailing list conversation][dtrace-marc] between Bryan
Cantrill and Rich Lowe that explains why this setup was necessary. It's awesome
that [marc.info](https://marc.info) kept a record of that conversation! To
overly simplify, collapsing the whole archive into one object file works around
the linker not naturally extracting archive members important to DTrace.
Remember the selective extraction behavior mentioned [earlier](#earlier)? As a
side note, while evaluating `ld -r` as an option for Rust integration, I ran
into an illumos [linker bug](https://www.illumos.org/issues/14722) when trying
to build Ruby without any Rust. Rich, from that mailing list conversation,
responded to my report and fixed the bug quickly. I felt like I was directly
interacting with computing history during that experience. Thanks, Rich!

Being able to work with a single object file instead of an archive also eases
some unrelated integration pain for YJIT. As a component of Ruby, YJIT needs to
fit into `libruby-static.a` --- it's not a separate library. To do this, we
used to
[merge](https://github.com/ruby/ruby/blob/v3_2_1/template/Makefile.in#L322-L330)
`libyjit.a` into `libruby-static.a`[^static-lib-merge-parallel]. This was a
build process complication specific to YJIT. On the other hand, the build
system already links many object files for C. Adding yet another object file to
the mix is a smaller, better rehearsed change.

Great, `ld -r` makes a self-contained `libyjit.o` out of the `libyjit.a` output
from Rust. Can we get Rust to output `libyjit.o` directly instead of getting it
from a partial link? There is `--emit=obj`, but it's intended for debugging
Rust itself. The output object from that option has references to Rust internal
libraries, so to link successfully we'd need to figure out a list of things to
pass to the linker, which themselves can end up leaking symbols. Also, upstream
[seems to not want you to rely on it as a stable interface][emit-obj-ng].

## Shared libraries and `-fvisibility=hidden`

I've been avoiding the verb "export" for describing the act of making some
symbol available for linking. This is because at the object file format and the
static library format level, there is not really a direct encoding for whether
a symbol is exported or not. When building a shared object, it's a bit more
clear. Mach-O dylibs encode all symbol exports in a [trie]; `objdump
--exports-trie` lets you inspect it. Simple. For ELF shared libraries, the
definition for an "exported symbol" is [a bit more complicated][elf-export].
Hmm, maybe avoiding the word wasn't a great idea.

Anyways, we also want to curate the exported symbols list when building a
shared library. Instead of build-time errors, shared library symbol conflicts
can show up as *runtime crashes*. That sounds worse.

A way do this with ELF and Mach-O is through symbol attributes. There is
"hidden" on ELF, and "private external" on Mach-O. With C
code, using `gcc` or `clang`, a way to make these symbols is through the
`-fvisibility=hidden` option, so I'll refer to these as "hidden symbols" from
here on out. In the final link that produces the shared object file, hidden
symbols are not exported. The idea with `-fvisibility=hidden` is that it makes
all symbols hidden by default, which then allows you select the few symbols you
wish to export by marking them as not hidden. It's an allowlist setup rather
than a blocklist one --- great for intentional curation.

Whether a symbol is hidden is a separate attribute from whether it is `extern`,
short for "external". From what we saw earlier, duplicate external symbols
cause conflicts when linking statically. Does making an external symbol hidden
solve the conflict? Let's try:

```sh
#!/bin/sh
# This is hidden-and-extern.sh.
# We were with ELF earlier, let's switch to Mach-O now.
# Run in an empty directory.
CC=clang
CFLAGS='-fvisibility=hidden -x c -std=c99 -c'

set -o errexit
set -o nounset
set -o xtrace

# First definition of h_and_e()
printf '%s' 'extern void h_and_e(void) {}' \
    | $CC $CFLAGS -o h1.o -

# Second definition of h_and_e()
printf '%s' 'extern void h_and_e(void) { __builtin_trap(); }' \
    | $CC $CFLAGS -o h2.o -

# Show symbol attributes for both definitions
# (this assumes you have a recent Xcode toolchain, which ships llvm-nm)
nm -f darwin h1.o h2.o | grep h_and_e

# Try linking to make a shared library
$CC -shared -o libmy.dylib h1.o h2.o
```

No, hidden external symbols conflict too:

```shell
$ sh hidden-and-extern.sh
+ printf %s 'extern void h_and_e(void) {}'
+ clang -fvisibility=hidden -x c -std=c99 -c -o h1.o -
+ printf %s 'extern void h_and_e(void) { __builtin_trap(); }'
+ clang -fvisibility=hidden -x c -std=c99 -c -o h2.o -
+ nm -f darwin h1.o h2.o
+ grep h_and_e
0000000000000000 (__TEXT,__text) private external _h_and_e
0000000000000000 (__TEXT,__text) private external _h_and_e
+ clang -shared -o libmy.dylib h1.o h2.o
duplicate symbol '_h_and_e' in:
    h1.o
    h2.o
ld: 1 duplicate symbol for architecture arm64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

On Mach-O, these `private external` symbols have both the [`N_PEXT`][N_PEXT] and the
`N_EXT` bit set --- they are separate attributes. Because `N_PEXT` sounds like
"private external", you may wonder what it means for a symbol to have `N_PEXT`
set, but `N_EXT` clear. Those symbols are in fact non-external, `N_PEXT`
[tracks][nm-pext] what the symbol once was:

```shell
$ echo 'int main(void) {}' | cc -fvisibility=hidden -x c -std=c99 - | nm -f darwin a.out | grep main
0000000100003fb0 (__TEXT,__text) non-external (was a private external) _main
```

Non-external symbols (`N_EXT` bit is clear) in the input to the final link are
naturally not exported. This makes sense considering the semantics of the
language features that produce these symbols. Think `static` functions in C and
private functions in Rust. These symbols also don't cause link-time conflicts
when linking statically.

Rust 1.67.1 doesn't provide a language feature for making hidden symbols. Also,
most external symbols defined by stdlib inside Rust's staticlib output are not
hidden. On Darwin, you can run `echo | rustc --crate-type=staticlib - && nm -f darwin --defined-only --extern-only --no-llvm-bc librust_out.a | less`
to see.
There are some `private external` symbols from `compiler-builtins`, a Rust
internal crate, but they are [built from C code using
`-fvisibility=hidden`](https://github.com/rust-lang/compiler-builtins/blob/5511f3087255236680eefb862458ed2f90e11bb5/build.rs#L179)!
This `-fvisibility=hidden` option wasn't available since day one; the first GCC
version to have it was 4.0.0, first released in 2005. Maybe Rust will gain a
similar capability in the future. [^2][^3]

CRuby used to support GCC 3, which doesn't have `-fvisibility=hidden`. To
remove undesired exports, CRuby used [`objcopy(1)`][objcopy] to
[post-process][objcopy-in-ruby] the `libruby.so` shared object after the final
link.

Maybe we can use `objcopy` to process `libyjit.o` too. Sure enough, there is
`--keep-global-symbol`, which instructs it to turn all but the global symbols
named by the option to be *not* global![^7] For our purpose of dodging symbol
conflicts, "globalness" in ELF is equivalent to "externalness" in Mach-O.

The invocation ends up being basically `objcopy --wildcard
--keep-global-symbol='rb_*' libyjit.o` since we already prefix all the symbols
we want to make available with `rb_`. This pretty much completes curation for
both Ruby configurations. For the static library configuration, the _hundreds_
of symbols Rust stdlib defines are turned non-external, leaving just a handful
of `rb_*` symbols we intend to make available.
For the shared library configuration, we also don't end up exporting Rust
stdlib symbols. The `rb_` symbols we define in `libyjit.o` _are_ exported because
they are not hidden, but that's acceptable. They're prefixed with `rb_` like
the rest of Ruby symbols and don't show in Ruby's public headers. We've shrunk
the surface for conflicts by a lot.


## Curating symbols on macOS without `objcopy`

As a part of GNU Binutils, `objcopy` is quite ubiquitous. It's not part of
Xcode Command Line Tools on macOS, though. There is no first party prebuilt
binaries for CRuby, so users build from source all the time. To not place yet
another tool installation burden on users, it'd be nice if we could curate
symbols without relying on `objcopy`. I'm already asking people to install Rust
if they want YJIT.

This is where I went combing through ld64's manual for anything relevant. There
is `-exported_symbol`, which seems very similar to `objcopy`'s
`--keep-global-symbol`, it even takes wildcards. Using a wildcard doesn't quite
work, though. It also modifies undefined symbols that `libyjit.a` references
which are defined in the C part of the codebase. When doing the final link,
each of these symbols it produces effective act as an assertion that there
exists an export with the same name. We use a lot of internal functions from
`libyjit.a` that are not exported. That won't fly.

I also tried a bunch of other option combinations. I should've taken better
notes as to what I tried, but nothing seemed to work. `nobu`, a longtime CRuby
maintainer, had [posted his
take](https://bugs.ruby-lang.org/issues/19255#note-1) on the problem on the bug
tracker, but it didn't pass macOS CI.

Thankfully, `nobu`'s take put me on the right direction. There were just a few
wrinkles to iron out.

### Combine and curate, all in one go

With `-exported_symbol '_rb_*'`, ld64 touches too many symbols. We only want to
manipulate defined symbols, a subset of what it selects. This is possible with
`-exported_symbols_list`. The manual talks about how global symbols _not_ in
the list are turned `private external`, which sounds not quite enough --- we
saw how these symbols can still conflict earlier.

But, `ld -r` turns `private external` symbols into conflict-free non-external
symbols! This feature is documented under the
[`-keep_private_externs`][-keep_private_externs] section in the manual.

The order of operations also works out in our favor. So, symbols not in the list
are turned `private external`, and then turned non-external. The exported
symbols list also helps ld64 extract the right archive members. One `ld -r
-exported_symbols_list` call gives everything we want. Slick! 

### Xcode's `nm` is `llvm-nm`

What list of symbols do we give to `-exported_symbols_list`? For YJIT, the list
is small enough that we could maintain it by hand, but then sometimes we'd need
to repeat ourselves when adding new Rust functions. `nobu`'s branch filters the
output of `nm --defined-only --extern-only` to generate the list. But that
errors out on CI:


> nm: error:
> yjit/target/release/libyjit.a(std-1a5555b33819f218.std.0c61b6a2-cgu.0.rcgu.o):
> Opaque pointers are only supported in -opaque-pointers mode (Producer:
> 'LLVM15.0.2-rust-1.66.0-stable' Reader: 'LLVM APPLE_1_1400.0.29.102_0')

Opaque pointer? I hardly know her! Well, I did vaguely recall that opaque
pointers are an LLVM thing. The manual for `nm(1)` mentions that it is actually
`llvm-nm` nowadays, and `llvm-nm` by default decodes the LLVM bitcode bundled
within the object file. Rust's LLVM is newer than Apple's, and Rust's bitcode
is using a construct from the future.

We're not doing fancy cross-language LTO here and only care about the machine
code in the object files, so we just need to pass `--no-llvm-bc`.

### The return of `rust_eh_personality`

Great, we can hide all the symbols we want. Let's try linking `libyjit.o` with
everything else...

```text
linking miniruby
Undefined symbols for architecture arm64:
  "_rust_eh_personality", referenced from:
      ___rust_try in libyjit.o
      ___rust_drop_panic in libyjit.o
      ___rust_foreign_exception in libyjit.o
      ...
ld: symbol(s) not found for architecture arm64
```

Excuse me? It's defined, and right there in the symbol table:

```shell
$ nm --no-llvm-bc --defined-only --format darwin yjit/target/debug/libyjit.o | grep rust_eh_personality
0000000000368534 (__TEXT,__text) non-external (was a private external) _rust_eh_personality
```

This seems to be an ld64 bug. LLVM's linker, [LLD], links it just fine. Is the
bug in `ld -r`? Well, I can recreate the error without any `ld -r` involvement:

```asm
; This is test.s
;
; ld64 fails to link unless the following line is uncommented:
; .private_extern _my_personality
;
; lld version 15 links successfully either way.
.align 4
_my_personality:
  ret

.global _fun
.align 4
_fun:
  .cfi_startproc
  ; 0x9b = DW_EH_PE_pcrel | DW_EH_PE_indirect | DW_EH_PE_sdata4
  .cfi_personality 0x9b, _my_personality
  ret
  .cfi_endproc
```

```shell
$ as test.s -o test.o && clang -shared test.o
Undefined symbols for architecture arm64:
  "_my_personality", referenced from:
      _fun in test.o
ld: symbol(s) not found for architecture arm64
```

I guess ld64 has special treatment for personality function references encoded
in [compact unwind]. Just a guess. Apple hasn't responded and for almost 2
years now, [hasn't released ld64's source code][ld64-since-2021]. I'm
discouraged to dig any deeper. [LLD] does not have support for `ld -r` for
Mach-O yet, but the error says to "stay tuned". Rust ships with LLD
already so I'm looking forward to when LLD adds support.

It feels like `rust_eh_personality` always turn up in [weird linkage issues].
Fine, we'll compromise and leave it exported for now. We hide all other
Rust stdlib symbols, though. Still a nice improvement.

## Closing

It's a bit rough plugging a small piece of Rust into a larger C codebase,
y'all. There are discussions[^4][^5][^6] upstream about improving various
aspects of the experience. Perhaps there is some opportunities for
contribution.

With respect to symbol leakage, I focused a lot on the problems in this post.
In practice, it may not be a high priority problem. The latest YJIT release
includes none of the mitigations discussed, and so far, we haven't received
reports of related build issues.

I remember benefiting from [posts](https://viruta.org/tag/librsvg.html) about
`librsvg`'s Rust and C integration when I first started looking into using Rust
for YJIT. I figure some might find what we do in YJIT  interesting. It was fun
sorting through all the pieces involved to make this post.

[^8]: You may know about ELF's `STV_PROTECTED` visibility. It doesn't do what
  I described.

[^static-lib-merge-parallel]: As a parallel, staticlib crates can statically
  link against native libraries, so Rust itself sometimes merges archives.

[^2]: Using hidden symbols is not the only avenue for avoiding undesired symbol
  exports. For Mach-O, ld64 provides the `-load_hidden` option to link against
  an archive without also exporting non-private external symbols in it. For
  ELF, GNU ld provides `--exclude-libs` for this purpose. These options are
  worth considering when linking against Rust staticlibs to produce a shared
  library. They don't help when making a static library, though.

[^7]: Of course, "local symbols" are not the opposite of global symbols...

[^3]: For `cdylib` crates, where Rust does the final link (not the case in
  YJIT), Rust started to trim exports starting in 1.62.0 thanks to
  [`rust-lang/rust#95604`](https://github.com/rust-lang/rust/pull/95604)! On
  macOS, you can see this by comparing `printf '%s' '#[no_mangle] pub extern "C" fn foo() {}' | rustc +1.61.0 --crate-type=cdylib - && dyld_info -exports librust_out.dylib`
  with a `+1.62.0` invocation.

[^4]: [Pre-RFC: Stabilize a version of the `rlib` format][formal-rlibs-pre-rfc]

[^5]: [Don't leak non-exported symbols from staticlibs](https://github.com/rust-lang/rust/issues/104707)

[^6]: [Formal support for linking rlibs using a non-Rust linker](https://github.com/rust-lang/rust/issues/73632)


[weird linkage issues]: https://github.com/rust-lang/rust/issues/102754
[LLD]: https://lld.llvm.org/
[-keep_private_externs]: https://github.com/apple-opensource/ld64/blob/e28c028b20af187a16a7161d89e91868a450cadc/doc/man/man1/ld.1#L406-L410
[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[Mach-O]: https://en.wikipedia.org/wiki/Mach-O
[objcopy-in-ruby]: https://github.com/ruby/ruby/commit/a2a53430330e8d010a479c6e85e9f2cf178efdf0
[objcopy]: https://sourceware.org/binutils/docs-2.40/binutils/objcopy.html
[nm-pext]: https://github.com/llvm/llvm-project/blob/86cbf3d5f8a208e1129b4d75383ef792f1d8a4aa/llvm/tools/llvm-nm/llvm-nm.cpp#L486-L507
[N_PEXT]: https://github.com/apple/darwin-xnu/blob/8f02f2a044b9bb1ad951987ef5bab20ec9486310/EXTERNAL_HEADERS/mach-o/nlist.h#L118
[elf-export]: http://www.m4b.io/elf/export/binary/analysis/2015/05/25/what-is-an-elf-export.html
[trie]: https://en.wikipedia.org/wiki/Trie
[formal-rlibs-gh]: https://github.com/rust-lang/rust/issues/73632
[formal-rlibs-pre-rfc]: https://internals.rust-lang.org/t/pre-rfc-stabilize-a-version-of-the-rlib-format/17558
[emit-obj-ng]: https://github.com/mesonbuild/meson/pull/11213#issuecomment-1366541458
[LTO]: https://doc.rust-lang.org/rustc/codegen-options/index.html#lto
[`rust-lang/rust#44322`]: https://github.com/rust-lang/rust/issues/44322
[CRuby]: https://github.com/ruby/ruby
[Hyrum's Law]: https://www.hyrumslaw.com
[ruby-dtrace-marc]: https://github.com/ruby/ruby/blob/31819e82c88c6f8ecfaeb162519bfa26a14b21fd/template/Makefile.in#L510-L512
[dtrace-marc]: https://marc.info/?l=opensolaris-dtrace-discuss&m=114761203210735&w=4
[illumos]: https://www.illumos.org/
[compact unwind]: https://faultlore.com/blah/compact-unwinding/
[ld64-since-2021]: https://opensource.apple.com/source/ld64/
