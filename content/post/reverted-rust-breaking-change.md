+++
title = "Hitting a Reverted Breaking Change in Rust"
date = 2026-06-07T00:00:01-04:00
+++

_First posted at <https://railsatscale.com/2026-04-20-hitting-a-reverted-breaking-change-in-rust/>_

The story starts with Rust 1.85.0 giving me a surprise `SIGTRAP` in some test code. For [reasons](https://github.com/ruby/ruby/pull/16647) not important here, I had to bump `opt-level` from 0 to 1 for tests. That made a bunch of tests fail with `SIGTRAP`, whose default signal handler terminates the process.

The `SIGTRAP` happened in Rust code code that call C.

## Passing a Rust closure through C code

The code that broke passes a Rust closure through a C function and invokes the closure in a callback.

The C API takes a user data argument in addition to the callback function, for passing through context:

```c
// Calls (*proc)(data). VALUE is uintptr_t.
VALUE rb_protect(VALUE (* proc) (VALUE), VALUE data, int *pstate);
```

Each closure has a distinct anonymous type as they can vary in size depending on what's captured. There is no way to separate out the captures and the function pointer to fit the C API. But, we can use a trait object to talk about a category of closure types, and the trait object has one uniform size:

```rust
fn closure_info(closure: impl FnMut()) {
    use std::mem::size_of_val;
    let trait_object: &dyn FnMut() = &closure;
    println!(
        "closure_size={} trait_object_size={}",
        size_of_val(&closure),
        size_of_val(&trait_object)
    );
}

fn main() {
    let mut int = 0;
    let one_capture = || int = 42;
    let no_capture = || {};
    closure_info(one_capture); // closure_size=8 trait_object_size=16
    closure_info(no_capture);  // closure_size=0 trait_object_size=16
    // Varying closure size, same trait_object_size.
}
```

The trait object is too big to fit the pointer-sized data argument, but we can solve that by taking a reference of it. That gives `&mut &mut dyn FnMut()`, a double reference to the trait object. `&mut` is pointer-sized, unlike `&mut dyn`. Great, we can now pass the closure through. Now we need to write the callback that invokes the closure.

### The transmute

To call the closure, we need to first turn the `VALUE` back into the double reference to trait object. I used `std::mem::transmute` for this, and it blew up.
Rust 1.85.0 compiles the following function to a single UD2 instruction which raises `SIGTRAP`.

```rust
#[repr(transparent)]
struct VALUE(usize);

extern "C" fn c_callback(obj: VALUE) {
    let closure: &mut &mut dyn FnMut() = unsafe { std::mem::transmute(obj) };
    closure();
}
```

Surprise `SIGTRAP`s like these from LLVM makes me think of trap mode in [UndefinedBehaviorSanitizer](https://clang.llvm.org/docs/UndefinedBehaviorSanitizer.html). What rules did I break?

## Probably pointer provenance

Integer to pointer transmute is [documented][transmute-doc] to have unspecified behavior, but our `struct VALUE(usize)` definition is not an integer. Like many parts of Unsafe Rust, it's hard to say how the transmute should have behaved. Rust 1.78.0 includes a change that seems to draw a clear line. ["Lower transmutes from int to pointer type as gep on null"][GEP] ("transmute patch" from here on) changes how `transmute` picks the provenance of the target pointer. Before, the transmute acted like an integer-to-pointer `as` cast, picking a previously exposed provenance. Now, it's based on the null pointer. The null pointer is invalid for access, and so is the derived pointer. The `SIGTRAP` is probably trying to say, in an obtuse way, that I'm calling an invalid function pointer.

I had a hunch this is related to pointer provenance changes. I also remember from discussions about provenance something about writing a signature on the Rust side that uses a pointer type to accept an integer argument from the C side. I'm not sure if it's a good idea to misrepresent types like this, but it's useful to do it as an experiment to see if the `SIGTRAP` goes away:

```rust
// Experiment: the C side calls this with an integer (VALUE), but we claim it's a pointer
extern "C" fn c_callback(obj: *mut ()) {
    let closure: &mut &mut dyn FnMut() = unsafe { std::mem::transmute(obj) };
    closure();
}
```

And this version doesn't `SIGTRAP`! So this is probably related to pointer provenance, rules that stipulate that in addition to having a good address, a pointer is valid for dereference only when obtained a certain way. The pointer in both the working version and the `SIGTRAP` version have the same address and in-memory representation, but only one is valid for dereference.
The `ptr` module has [documentation](https://doc.rust-lang.org/core/ptr/index.html#provenance) about provenance rules.

Searching in `rust-lang/rust` found me the transmute patch, and [experiments][experiments] on Compiler Explorer with various Rust versions showed 1.78.0 to be the first version that compiles the function to a single UD2 instruction. It's the first release that includes the transmute patch.

But why does it `SIGTRAP` only on older Rusts? What changed? 

## Revert due to LLVM provenance bug

In the `rust_has_provenance` [RFC][RFC], the lack of proper treatment of provenance in LLVM is stated as a drawback. Turns out, the transmute patch triggers such bugs and so was [reverted][revert] in version 1.91.0. In some situations, the existence of a pointer with invalid provenance in the system can have LLVM confused and wrongly decide that an unrelated pointer is invalid for access. To be clear, the LLVM bugs are not the reason my code raises `SIGTRAP`; the transmute patch is designed to break such code. The unintended breakages from the transmute patch were due to the LLVM bugs giving bad output for code completely absent of Unsafe Rust.

There are many reports of code breakages due to the transmute patch, but the Rust team did not consider it to be a breaking change. I think it deserved a place in the release note as a compatibility issue. It also would have been nice if it panicked with a clear message rather than a nondescript `SIGTRAP`, but maybe the UD2 comes from a place too deep in LLVM's pipeline to realistically replace.

With the revert, the `SIGTRAP` is gone. Absence of problematic runtime behavior does not imply absence of [Undefined Behavior][UB], though, and the revert did not change rules of the language. The current documentation for `std::mem::transmute` is clear about this subject:

> Transmuting integers to pointers is a largely unspecified operation. It is likely not equivalent to an as cast. Doing non-zero-sized memory accesses with a pointer constructed this way is currently considered undefined behavior.

Let's try to follow the rules.

## The fix

Using an `as` cast instead of `transmute` [avoids][fixed] the `SIGTRAP` in all Rust versions:

```rust
extern "C" fn c_callback(obj: VALUE) {
    let closure = obj.0 as *const *mut dyn FnMut();
    unsafe { (**closure)() };
}
```

I understand why `as` cast works here better than `transmute` as follows:
`transmute` works solely with the bits of the input value, but provenance is not represented in the input integer (integers have no provenance),
so the output pointer has no provenance and is invalid for dereference. On the other hand, `as` casts can add things to the output not in the input. For casting from a smaller integer size to a larger one, it adds bits. Here, it adds provenance.

## Ready for round two

I wrote code that triggers [Undefined Behavior][UB]. We fix the code and move on, having learned a bit about pointer provenance language rules. If and when the transmute patch is reintroduced after the LLVM bugs are fixed or mitigated, our code will work.

[GEP]: https://github.com/rust-lang/rust/pull/121282 
[RFC]: https://rust-lang.github.io/rfcs/3559-rust-has-provenance.html
[UB]: https://doc.rust-lang.org/reference/behavior-considered-undefined.html
[revert]: https://github.com/rust-lang/rust/pull/147541
[transmute-doc]: https://doc.rust-lang.org/1.95.0/std/mem/fn.transmute.html#transmutation-between-pointers-and-integers
[experiments]: https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyahSAVyVFyKxqiIEh1ZpgDC6embZMpAJnLOAGQImbAA5TwAjbFIQAFZyAAd0JWIHJjcPL19E5NShIJDwtiiY%2BJtsOzSRIhZSIgzPbx4/csqhatqiArDI6LjrGrqGrOaBzu6ikriASmt0M1JUTi4AUh8AZhXYgCFSbATSCCJSYyUE2uYiaa2AERWtAEFLczsAagA1B4CFXAgLAgAXthruttvcHuC1psdkx0AB9NjGYCMW7g7DqIjRJivNY%2BFy4140bGoOHiej0CIsVAAawg6Ai%2Bg%2BXx%2B0xxAHYwY9XtzXowiK9UPRkgtODifAA2NhmflrSXS16YACe2IAYkwALLSiCslbrG6vMxMJQsGjYdnbV6WTAgEAcNg246nKWYukMnVsu6g8E8gVCpQi7W6zkQj1cWb0bixfjeLg6cjobgAJQs/KU80WZqhfHIRG0Ydm1LiWkM3Gk/Ht61iADoABxaNk%2BACcPHFrbb4rZ5BjcYTXH4ShAxdzsbD5DgsBQGDYCQY0Uo1CnM8YMVIPDZbOLdHomNIA4gETz5AiwVqiu42ePrFIioA8gyKsPs1OOMIb0x6GeR%2BQcBEzMAXBI9ADrw/A4IiJiSF%2BhB7JUABu2DAXG6IVNKyxxsEmIRl%2B9AEBEJzXm4OCHscBD2iB5DwaQETJNgNzYOByLBKAI6zDQRjAEo7wENgADuN4JMw578IIwhiBInAyHIwjKGomhfvofhGCYIDmJYhi4QOkCzOgCT2EIwEALQGdghB6Uwuo3D4Wg%2BDwrwGW4ukGYw8H0BZ2boJRpAEDgmlQKwHAgCZeRMBREhmMsVk%2BGW0yzK0ZlOEwrjuI0IA%2BPEgTBD0xR9DwxZJCkZnDN4aU5AVaQTL0MS5dY2C2GZHRDMlWQlXFVSDF0mWTDlxaWJ0RWpWU7UVdlVVaLF6ZLFI4aRtGh69q8qlEKgrw8FW65VlorwQPgxBkGK6w8NM/DDjoMXkEg2AsDgMTaiWXBluQFbilWPhsjINaNq2dY2eKNZdnN3D9oOOZ5rM47IGg6DTrOFBUBAi4wypa51gIDA7nuB5fpep5CUeJ7XneugPrjz6XG%2BH6Hj%2Bf4AeSwHZmBSKQXG0EPgQ8GIfwyGoKhh4YbVh44Xhp6EWhx1eWR2aUdRKh0QxOHKSxAjsZx3F8QJMbZiJohkhJshazJGiHvo6yGEiKkpupES%2BdpulpIZxmmWkFmRbZ9k6UQTnYC5bn8B50ReT58AQP5nBBWZoUeBF1nRbFtWs44EDOP1PD%2BIlw1TCn%2BXBcnpXBenOU1XVbV9U1TSF/HTANR1hSVQYvWNZkZf19XWUZ%2BNCyTYdd1Rv9X7zYty2rWyNYbVtO0kKQ%2B2HcdoOzBdV19LdWEPfar1Vo2Nk2TWraxAdLbxN2vuA9YwMnfm5CFrExZYess198fZ9g4g4MQJOUNLnOcMI8uSPNpuaPRAxoebG15cYgNvPeOwJMoYviIOTT8zNsC/n/IBOmoF6KM1Ft%2BAgMF7Ds0PFzHmX4%2BZYTjILfCioRbEXFrjKWNFZaMyYqDJWLAOJcV4vxQS5EtZiUkJJfWKhDbyQMEpUwFtBbW3jLbfS3AjJhydnqF2dkHIe2crVH28ZPLeQQkHEOgVHZCAjuFbgkUY7lzaN4ROiUc4ZRriNAwWdCqlwcbkMy%2BcqrmPqu1HOrV2hDU6rXFOzcfH%2BLsW3OYHcJLTS4D3Q%2B8ZuALRTIPdeq1NrbVMntLMR0QYsQLEWO6K98lxN7EDIczCX5v2hsuec8N36I1XI2LQ/9tyAOoJjOM4CwH4wgUTKB5FSavnfAg9ByCaZAVxgzCCWCWZwR0V%2BQhmJebCH5thXCFCqFfhIhLfgdCZYYIgkwxWbFWEqw4erXGPCdZSD1vIA2ck4z6FkKI82akJHwBtmZe28ihDO2sq7FRntvZ6nclowOsA9HfJCrBMKUcbLn18ZYpOzieAp1sa3Pof1HFpGTpnVx5UAn2L%2BgiquOLPHFzqO4kARLvHIqCaE9FMQaztwzFNbud8ewJIHitdeWhR7pN2pPLJM9ckFPLEUgGfYT5lJFZfa%2B3Bb69w5ZKx%2B0SfDsqPsq2eFFAFpBANIIAA%3D
[fixed]: https://godbolt.org/#z:OYLghAFBqd5TKALEBjA9gEwKYFFMCWALugE4A0BIEAZgQDbYB2AhgLbYgDkAjF%2BTXRMiAZVQtGIHgBYBQogFUAztgAKAD24AGfgCsp5eiyahSAVyVFyKxqiIEh1ZpgDC6embZMQANgBM5M4AMgRM2AByngBG2KQgsgAO6ErEDkxuHl6%2BAUkp9kIhYZFsMXGyNth2aSJELKREGZ7e/tbYtvlMNXVEhRHRsfHWtfWNWS2W3b3FpfEAlNboZqSonFwApH4AzGsArABCpNgJpBBEpMZKCXXMRLO7ACJrWgCCluZ2ANQAas9BCrgQCwEABe2Dumz2T2eUIA9DCPmsfFpLJgQCAEmc0QB3YhIAD62HUuWwmDxx3QADdmMYVoitB8mOgiB9jtgVMIPqEPjwAHQAdj5PK0PKhG22%2B0ZeLYxmAjAeUMJRFiTARfj8Lg2fg%2BNBVqDx4no9CiLFQAGsIOgovpvr9/rMEXzIS8Pi6Poxmah6MkltgEZt7h9LbohR8WEoPgAqDBMSyRthmZmYACeKoAYkwALIJiDgp3PV0fMwxlg0X1rR0fCARqNepQ%2B2Y5h2PCGivn3LjzejcHb8bxcHTkdDcABKFmZSkWyzLWz45CI2g781NIB2WkM3Gk/DYK7XfYHQ64/CUIDX8/7HfIcFgKAwbASDFilGot/vjDipB4ArXdHoStIx4gKIF3IKJQjqJNuFnUDWFIJMAHkrUqM9Z1vDhhDgph6Ag89yBwKIzGAFwJHoY9eH4HBpRMSQcMIQ4qipUiB0JSoE1WAdQiVLscPoAgonOWC3BwYCzgIbcyPIKlSCiZJsHubBKNlUJQHPeYaCMYAlC%2BAhsCxOCEmYSD%2BEEYQxAkTgZDkYRlDUTQcP0AIjBMEBzEsQxeOPSB5nQDE0lIgBaPySVSIQ1n9PwtD8HgPj8twMT8xgqXoUL7lnSlYlIAgcA8qBWA4EAgo6CSJDMVZwr8TdZnmCoqkcCBnFGbweECJhMCmfo4ia3JgvSdwmgMLqOjakoBia6qOi6EZeqyUa2iQ6phh6UI%2BmGjqhm6BqDAmeohpmHgqsnFYpE7bte2Ag8PhcohUG5flBXpCB8GIMhVU2Pb%2BDPHRKvIJBsBYHA4hzdcuE3chtz8HZyD3fgDyPE85wXeYr2QNB0DvB8KCoCAX3R5zPwADm/Bg/wAoCcOg8DDJAsDYIQ3QkMp1CbgwrDgLwgiiMNUjZwomVqIHWi5oY4DmNQVjgI4tpgJ4vjwMEtj3oysTZ0k6SVDkhSeKclSBHUzTtN0/S%2B1nYzRANczZBN6yNGA/RNkMGVnLHNyomyryfKEfzAsIDpkrKqKYu8oh4uwRLktSySMqy%2BAIFyzgCrSIqPFKiKKqq2aau8OqWo2prgiW6YRvIAa0hzovkm6nbC7G%2Bb1qmxrWnaGvtvz9rNoW0utsWopW72hYlkO3uuJ7SGzu4C6x2u3k%2BTxkMHu956xTe%2BGVPmH6/oGQGuJBsGIahwduFh08EaXHcgc2U6cJh5fPsvRAkYgG9UdfR9Mext9cYATh4Qnf1iEngPJrBSmgD4KITsAzVGaEiDM2wvzbA%2BFCLES5uReSvN5a4QIHRewQscIizFjhCWXEBzS34kmOWwlFaUxVjJdWvMlIIx1iwDSWkdJ6QMuJE2plJAWUtioa2dkDCOVME7aWrtBzuxjNwAK8cQphQiv7WKQcEptDDtDCOmU2TR1jvlb2CcKTFWTuVRcDc5q1XqnXAwedu4rX6uXDopdi4FBbjYmajchATQaBY1xpjOgLUrqtTuHc/HON2vtfu5ljpcGHnvc6l1J48i/rPR6JBSAvSXh9Yxy5VxA23qfGJB9rBwwyYjO%2Bj80ZvifFjJ%2BOMPwfy0D/Ym1BSYDhAcA6moC6bgPEozdCmFYEoIQRzEilMeZUXQQLeimjcHqBYkqcWwhJbcV4qQ8hOERJK34NQtWqCqL0O1mpJhetWGG0ppws2UgLbyCtrZAc%2BhZBCMdq5UR8A3YdE9jIpgvt5HRUUcHUO/pw7pQ0dlGO7A466KEInEq3A/bGOrmY7OXjmqtRCYXRxPVMj13Rf4za6dxrtyRfC3xkxUUBIJZituJLrGhL7lOI6QNomjy4OPSw8S6lJPnqkxesx3rHxyVuPJTLD7X0yafLi58R6XwKcUyJfgL77mlXyySKRHDSCAA%3D%3D
