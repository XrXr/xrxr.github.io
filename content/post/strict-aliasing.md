---
title: "How I Follow Strict Aliasing Rules: A Case Study"
date: 2022-04-24T17:33:13-04:00
draft: true
---

Recently I was asked about how to review C99 code for problems that
arise from failing to follow the so called "strict aliasing rules".
I struggled to present my thought process while vetting code for this class
of problem, so I thought I would write a post to hopefully make my
explanation more coherent.

Just a disclaimer, I don't claim that my mental model is 100%
correct and exactly what happens during compilation. This model has helped
me spot [problems in code] and has given me enough confidence to spot
[problems in the compiler], but at the end of the day it's an abstract model from someone
who doesn't work on the compiler that actually runs. Better than nothing!

The strict aliasing rules can be surprising because the way optimizers take
advantage of them doesn't mesh well with the popular belief that pointers are "just numbers".
Ultimately, I think there are practical benefits for understanding the rules even if you disagree
with them since popular compilers such as GCC and Clang take advtange of the rules.

# What are the rules, anyways?

The relevant part in C99 is [§6.5p7], but in my head it
basically boils down to "two value accesses are disjoint when the types are different,
except when one of the types is a `char` type". Yes, this throws away some subtleties and it's
not going to get me into [WG14].

What happens when the optimizer can see that a write is disjoint
with respect to a read? It can decide to reorder the program
and do the read first if it seems profitable for performance.

Here is a [sample](https://godbolt.org/z/xM6fxb9or) where we can see GCC making use of the aliasing rules:

```C
unsigned
reorder(unsigned *foo)
{
    *foo = 0u - 1u; // All bits one. Unsigned arithmetic wraps around.

    short *ptr = (short *)foo;
    *ptr = 0;

    return *foo;
}
```

Compiling with `-O2` gives the following:

```asm
reorder:
    mov     DWORD PTR [rdi], -65536
    mov     eax, -1
    ret
```

In two's complement encoding, `-1` is all bits one. If we successfully write zero to any
part of an all bits one number, surly the result woud _not_ stay as all bits one!

GCC's output has a different order than our source program; `*ptr = 0` seems to have no effect on the
final read from `foo`, even though one might understand `foo` and `ptr` as
having the same address and expect `*ptr = 0` to happen first like in the source code.

Compile with `-O2 -fno-strict-aliasing` and we get something different:

```
reorder:
    mov     DWORD PTR [rdi], -65536
    mov     eax, -65536
    ret
```

Ah ha! By default, when GCC gets to use the power granted to it by the standard, it can assume
that the `short` accesses have no effect on `unsigned` accesses,
but `-fno-strict-aliasing` tells GCC to forget about that part of the standard.

The [bug report guide for GCC](https://gcc.gnu.org/bugs/) has a section about about `-fno-strict-aliasing`,
maybe because many people have been caught off guard by this optimization:

> To disable optimizations based on alias-analysis for faulty legacy code, the
> option `-fno-strict-aliasing` can be used as a work-around.

Oof. Okay GCC, type-based alias analysis is great and useful, but no need to judge this hard.

# Snap back to reality

Let's go look at a real-world example in CRuby where we failed to follow the rules.
This example has to do with an out parameter, where we expect the function to do a write
using the out parameter before returning.


aside: This is with `./configure --disable-install-doc -C cflags=-flto LDFLAGS=-flto` with GCC.

```text
vm_insnhelper.c:4948:9: warning: ‘ics’ may be used uninitialized [-Wmaybe-uninitialized]
 4948 |         st_insert(ics, (st_data_t) ic, (st_data_t) Qtrue);
      |         ^
vm_insnhelper.c: In function ‘vm_ic_compile_i’:
vm_insnhelper.c:4942:19: note: ‘ics’ declared here
 4942 |         st_table *ics;
      |                   ^
```
[§6.5p7]: http://port70.net/%7Ensz/c/c99/n1256.html#6.5p7
[WG14]: http://www.open-std.org/jtc1/sc22/wg14/
