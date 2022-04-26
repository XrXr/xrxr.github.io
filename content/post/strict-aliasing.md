---
title: "How I Follow Strict Aliasing Rules: A Case Study"
date: 2022-04-24T17:33:13-04:00
draft: true
---

Recently I was asked about how to review C99 code for problems that arise from
failing to follow the so called "strict aliasing rules". I struggled to present
my thought process while vetting code for this class of problems, so I thought
I would write a post to hopefully make my explanation more coherent.

Just a disclaimer, I don't claim that my mental model is 100%
correct and exactly what happens during compilation. This model has helped
me spot [problems in code] and has given me enough confidence to spot
[problems in the compiler], but at the end of the day it's an abstract model
from someone who doesn't work on the compiler that actually runs. Better than
nothing!

The strict aliasing rules can be surprising because the way optimizers take
advantage of them doesn't mesh well with the popular belief that pointers are
"just numbers". Ultimately, I think there are practical benefits for
understanding the rules even if you disagree with them. Popular compilers such
as GCC and Clang take advantage of the rules so knowing them can help with
debugging if for nothing else.

# What are the rules, anyways?

The relevant part in C99 is [§6.5p7], but in my head it basically boils down to
"two value accesses are disjoint when the types are different, except when one
of the types is a `char` type". Yes, this throws away some subtleties and it's
not going to get me into [WG14].

What happens when the optimizer can see that a write is disjoint with respect
to a read? It can decide to reorder the program and do the read first if it
seems profitable for performance.

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
part of an all bits one number, surly the result would _not_ stay as all bits one!

GCC's output has a different order than our source program; `*ptr = 0` seems to
have no effect on the final read from `foo`, even though one might understand
`foo` and `ptr` as having the same address and expect `*ptr = 0` to happen
first like in the source code. Adding to the surprise, GCC has combined the two
indirect writes into one, seemingly via the understanding that `foo` and `ptr`
have the same address! There seems to be some strange contradiction.

Compile with `-O2 -fno-strict-aliasing` and we get something different:

```
reorder:
    mov     DWORD PTR [rdi], -65536
    mov     eax, -65536
    ret
```

Ah ha! By default, when GCC gets to use the power granted to it by the
standard, it can assume that `short` writes have no effect on `unsigned` reads,
but `-fno-strict-aliasing` tells GCC to forget about that part of the standard.

The [bug report guide for GCC](https://gcc.gnu.org/bugs/) has a section about
about `-fno-strict-aliasing`, maybe because many people have been caught off
guard by this optimization:

> To disable optimizations based on alias-analysis for faulty legacy code, the
> option `-fno-strict-aliasing` can be used as a work-around.

Oof. Okay GCC, type-based alias analysis is great and useful, but no need to judge this hard.

# Snap back to reality

Let's go look at a practical example in CRuby where we failed to follow the
rules. If you'd like to follow along, you can grab this [commit] and build with
the following commands:

```shell
$ ./autogen.sh
$ ./configure cflags=-flto LDFLAGS=-flto
$ make -j8 miniruby
```

I'll be using GCC 11.2.0 on a GNU/Linux distribution.

This example has to do with an out parameter, where we expect a function to do
a write using the out parameter before returning. The call site looks like
this:

```C
typedef VALUE unsigned long;
typedef ID unsigned long;
typedef struct st_table st_table;

int rb_id_table_lookup(struct rb_id_table *tbl, ID id, VALUE *valp);
//                                                            ^^^^
//                                                  out param of interest

void
do_lookup(struct rb_id_table *const_cache, ID id)
{
    st_table *ics;

    if (rb_id_table_lookup(constant_cache, id, (VALUE *) &ics)) {
        // successful lookup
        st_foreach(ics, iterator_fn, 0);
    }
}
```

When `rb_id_table_lookup()` returns 1, it indicates that it has written through `valp`:

```C
//... inside rb_id_table_lookup()
if (index >= 0) {
    *valp = tbl->items[index].val;
    return TRUE;
}
else {
    return FALSE;
}
```


(*** TODO not happy with this paragraph. How do I show that there is a write and
a read of mismatching type simpler?)
Sorry for throwing code at you; I'm trying to give enough context so we can
pretend to be the compiler ! We are interested in the code path where the
lookup succeeds and we are inside the `if` block in `do_lookup()`. There
are really just two accesses, a write to `VALUE` and a read from `st_table *`.


```text
vm_insnhelper.c:4948:9: warning: ‘ics’ may be used uninitialized [-Wmaybe-uninitialized]
 4948 |         st_insert(ics, (st_data_t) ic, (st_data_t) Qtrue);
      |         ^
vm_insnhelper.c: In function ‘vm_ic_compile_i’:
vm_insnhelper.c:4942:19: note: ‘ics’ declared here
 4942 |         st_table *ics;
      |                   ^
```
[§6.5p7]: https://port70.net/%7Ensz/c/c99/n1256.html#6.5p7
[WG14]: https://www.open-std.org/jtc1/sc22/wg14/
[problems in code]:
[problems in the compiler]: 
[commit]: https://github.com/ruby/ruby/commit/697eed63e81eff0e02226ceb6ab3bd2fd99000e3
