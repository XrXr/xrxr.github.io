---
title: "How I think about C99 strict aliasing rules"
date: 2022-05-23T22:31:37-04:00
---

Recently I was asked about how to review C99 code for problems that arise from
failing to follow the so called "strict aliasing rules". I struggled to present
my thought process while vetting code for this class of problems, so I thought
I would write a post to hopefully make my explanation more coherent.

The strict aliasing rules can be surprising because the way optimizers take
advantage of them doesn't mesh well with the popular belief that pointers are
"just numbers". Ultimately, I think there are practical benefits to
understanding the rules even if you disagree with them. Popular compilers such
as GCC and Clang take advantage of the rules so knowing them can help with
debugging if for nothing else.

This is just my simplified model for how C compilers make use of the rules, and
I don't claim that it's 100% correct. However, I have found it useful enough to
spot [problems in code][clang-arm-tbaa] and when the code looked fine, used it to spot
[a problem in the compiler].

# What are the rules, anyways? {#rules}

The relevant part in C99 is [§6.5p7], but in my head it basically boils down to
"two value accesses are disjoint when the types are different, except when one
of the types is a `char` type". Yes, many subtleties are out the window and it's
not going to get me into [WG14], but I think it's a useful level of understanding
regardless.

What happens when the optimizer can see that a write is disjoint with respect
to a read? It can decide to reorder the program and do the read first if it
seems profitable for performance.

Here is a [sample] where we can see GCC making use of the aliasing rules:

```C
unsigned
reorder(unsigned *foo)
{
    *foo = 0;

    short *ptr = (short *)foo;
    *ptr = 1;

    return *foo;
}
```

Compiling with `-O2` gives the following:

```asm
reorder:
    mov     DWORD PTR [rdi], 1
    xor     eax, eax
    ret
```

The code is using an idiom to zero out `eax`, but the gist is that it's returning zero.
GCC's output has a different order than our source program; `*ptr = 1` seems to
have no effect on the final read from `foo`, even though one might understand
`foo` and `ptr` as having the same address and expect `*ptr = 1` to happen
before `return *foo`, as ordered in the source code. Adding to the surprise, GCC has combined the two
indirect writes into one, seemingly via the understanding that `foo` and `ptr`
have the same address! There seems to be some strange contradiction.

Compile with `-O2 -fno-strict-aliasing` and we get something different:

```asm
reorder:
    mov     DWORD PTR [rdi], 1
    mov     eax, 1
    ret
```

Ah ha! By default, when GCC gets to use the powers granted to it by the
standard, it can assume that `short` writes have no effect on `unsigned` reads,
but `-fno-strict-aliasing` tells GCC to forget about that part of the standard.

GCC is organized as optimization passes and separate passes don't necessarily
share information. The strange inconsistency we saw when we compiled with default
options is likely a result of this -- the `mov` and the `xor` are likely coming from
two separate parts of the compiler that don't share the same understanding of our
program.

The [bug reporting guide for GCC](https://gcc.gnu.org/bugs/) has a section
about `-fno-strict-aliasing`, perhaps because many people have been surprised
by this optimization:

> To disable optimizations based on alias-analysis for faulty legacy code, the
> option `-fno-strict-aliasing` can be used as a work-around.

Oof. Okay GCC, type-based alias analysis is great and useful, but no need to judge this hard.

# Snap back to reality

Let's go look at a practical example in CRuby where we did not follow the
rules. If you'd like to follow along, you can grab this [commit] and build with
the following commands:

```shell
$ ./autogen.sh
$ ./configure cflags=-flto LDFLAGS=-flto
$ make -j8 miniruby
```

I'll be using GCC 11.2.0 on a GNU/Linux distribution.

This example has to do with an output parameter, where we expect a function to do
a write using the out parameter before returning. The call site looks like
this:

```C
typedef unsigned long VALUE;
typedef unsigned long ID;
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

Let's focus on the code path where the lookup succeeds and break it down into
a sequence of accesses by type:

1. write `unsigned long` aka `VALUE` through `valp`
2. read `st_table *` using the `ics` local variable

Uh oh, `unsigned long` and `st_table *` are distinct types, so by the [aliasing
rules](#rules) the compiler is free to assume that the two accesses have no relation. If it decides
to reorder and do the read before the write, that would betray our
intention -- we want to make use of the output from the successful lookup so
we always want the write to happen first!

Does GCC tell us anything about this mismatch between our intention and what we wrote?
Why yes:

```text
vm_method.c:146:9: warning: ‘ics’ may be used uninitialized [-Wmaybe-uninitialized]
  146 |         st_foreach(ics, rb_clear_constant_cache_for_id_i, (st_data_t) NULL);
      |         ^
class.c: In function ‘clear_constant_cache_i’:
vm_method.c:143:15: note: ‘ics’ declared here
  143 |     st_table *ics;
      |               ^
```

That's a bit of a strange warning to get if you expect accesses to happen in source
code order. I suspect what has happened under the hood is that GCC _considered_
putting the read before the write and while evaluating that schedule GCC detects that
it reads from an uninitialized variable. I think GCC only sees this read-before-write schedule
when interpreting the aliasing rules strictly because adding `-fno-strict-aliasing`
makes the warning disappear.

The fix for this issue makes the code write and read through the same type. If you're
in the mood for an exercise, you can imagine what the code change looks like before
looking at the [patch](https://github.com/ruby/ruby/commit/5c61caa48154e3e43ff29ab865310aa9bdd9e83a).

# Takeaways

This post tries to build intuition for spotting strict aliasing issues.
The analysis I showed involves distilling the program under review into accesses by
type, sort of like taking a projection of it.
The CRuby example is _interprocedural_ and to
get all the requisite information for our analysis we needed to reference two
functions in separate files. Similarly, GCC issues a warning about
the code only when we build with [link time optimization](https://gcc.gnu.org/wiki/LinkTimeOptimization),
where it can reason about the two functions in separate translation units together.

Have fun coding in ISO C and be careful casting pointers!

[§6.5p7]: https://port70.net/%7Ensz/c/c99/n1256.html#6.5p7
[WG14]: https://www.open-std.org/jtc1/sc22/wg14/
[clang-arm-tbaa]: https://marc.info/?l=ruby-core&m=161463889519092
[a problem in the compiler]: https://gcc.gnu.org/bugzilla/show_bug.cgi?id=101868
[commit]: https://github.com/ruby/ruby/commit/697eed63e81eff0e02226ceb6ab3bd2fd99000e3
[sample]: https://godbolt.org/#z:OYLghAFBqd5QCxAYwPYBMCmBRdBLAF1QCcAaPECAMzwBtMA7AQwFtMQByARg9KtQYEAysib0QXACx8BBAKoBnTAAUAHpwAMvAFYTStJg1AB9U8lJL6yAngGVG6AMKpaAVxYMQAZi6kHAGTwGTAA5dwAjTGIQADZSAAdUBUJbBmc3D29fROSbAUDgsJZI6LjLTGtUoQImYgJ0908fC0wrPIZq2oIC0Iio2IsauobM5oUh7qDe4v6YgEoLVFdiZHYOVwZk4GD0AFINAEFiTBIsYggNrZ2AagAqflQ5/YPdgHYAIWfr77uH692vAARa4aAGfQ5fH4KBAkAh3eIEYj/IHXCDQ2F3OYPMGQ763BFIgHArg4iGHH7XY4EZYMX6oVCkl6vQHPDgLWicACsvE8HC0pFQnEc1wUSxWmH%2BACYfLwCJo2QsANbeDQAOg0AE5XhoAByvLyvHUxSQaQ1xDkcSQ8%2BUCzi8BQgDSkOV8tmkOCwGBQL0QJBoFjxOhRciUf2B%2BjRZDALhcLx8OgEKIOiDhG3hIK1ACenB4pH9bEEAHkGLRs67SFgWIZgOJy/hjpUAG6YB3lzCqCquRM53hBRMW/m0PDhYhZ5xYG2IvAsHsLKgGYAKABqeEwAHdC/FGD2ZIIRGJ2FJd/IlGobbpJfpqyBTMZzEPwg7IAtUAjUq2ALSFyXXD/jPZAsgGoar%2BW7oIYNjIL%2BADqYi0DB4zEHg1gfmIeBMMkRhErm5SVHYEAOCMni%2BAEUxFCUeg5CkAhEZRSTUQwPTkf0vi4e0nTDC4jR6GxVQTExfTRKxEy0cJXQCTMQkLKKyyrHoiKYGsPDsly1rloKHCqEaH7GtcwDIFBMaql4qKOKQ1y4IQJBSs01zOAGQaEtKkpzLK8pzAsCCYEwZyUEqICcpIxmBV4IVhZIoX6JwVqkDOMSSqqnKkLy/IafajrOu57o%2BkgSwEPEXYhhAYaOSErBrFpMQ6ZIekGdcRleLwmD4EQSHoHo/B7qI4hHp1J4qOo5YXqQa6jvEs5RRw3LJTaGmFl2BVwqgVDXJV1W1YZXDGaZqAORGNkuW5roeaQXk%2Bf0ED%2BZInKqsanJ3Q910xElFoxSlvBpRYGUuloJ0WpKvAzpyOrGV4YPgxD4NxO9tocEdv3ZfAHqICgu3hsGFDFWjjkoNGsbxrQibEMmqblumzDEGWub5owBDFqWNqVtWtb8vWFQ2M2rb8u2nbdtwvaCK0NoPqOlPjms/JTjO/Nzguy6rhuW68rmfX7j10h9YoA3nnoBhGDeZj6MOT6XQKb4CJ%2B36/lQDCoH%2BU4oWhGFBMAv7/kSQEgR%2BYEQchMFwQhDsEKhQ7O1hQI4a07OpPYDBOFxmQkXHEkUdk9HtKJCTp6kKcsS0bR8V0me8QIHGTIUgk8SJCfEYM4lkZXXDSWKcm%2BApSluha00wxpa26fpm3bRAZkWS11m7NKvh2dj%2B0Tz4rmZcdnneb5pvKoFiUaNdW%2BcjvO%2BTTFM6SFtMSn2f59n68M3qXaX1Oj9brI76IB5YtRUlRGZVsJwfc1QP9VbUan4MebUOqyDVoeDWsgtZniGnoUaTBxoy0mt3WanB5r5S7NcZaq1tL9zqg1Hae0og2SbvDBUp0V4XSujdSQGpAr0LoQwzkL1opqVSrfB0993IqQ4ADWKIBj63QviImIV8e63wfidZUMQNSqg1BoSUGouCSnirGTkGpJSsI4EAiRcNF6/V4fwvR5CTrNmJjHSQQA%3D%3D

