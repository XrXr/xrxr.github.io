---
title: "ARM64's MOVN instruction is clever"
date: 2025-09-16T00:00:01-04:00
---

The bit twiddling office called. They want a function to compute a sequence of
ARM64 instructions that fills a register with an arbitrary constant 64-bit
number.

## Zeros in positive number

All ARM64 instructions are 32 bits long, so one instruction can't possibly
handle all 64-bit patterns. One straightforward way to assemble the desired bit
pattern is to use
[`MOVK`](https://developer.arm.com/documentation/ddi0602/2025-06/Base-Instructions/MOVK--Move-wide-with-keep-?lang=en)
which sets any one of the four 16-bit chunks of the register without touching
any other chunks. So, four `MOVK` instructions can assemble any 64-bit pattern.

Adding `MOVZ` to the mix improves this strategy. Small numbers have many zeros
in the pattern, so we'd rather start by zeroing the whole register than
zeroing out one 16-bit chunk at a time with `MOVK`. In one go, `MOVZ` sets one
16-bit chunk to the desired value and also zeros out all other chunks. For
small numbers like 7, all we need is one `MOVZ`.

## The negative ones

Another instruction in the same family is
[`MOVN`](https://developer.arm.com/documentation/ddi0602/2025-06/Base-Instructions/MOVN--Move-wide-with-NOT-?lang=en).
It sets the register like `MOVZ`, but with all bits inverted. So, `MOVN` sets
the three chunks that are not directly encoded in the instruction to all ones.
Why would you want so many ones? Negative numbers!

The most significant bit of a two's complement number contributes negatively to
the value tally unlike the rest of the bits. Adding a leading one to the bit
string makes the old most significant bit at position K go from having a value
of {{<html>}}-2<sup>K</sup> to 2<sup>K</sup>, adding
2•2<sup>K</sup>=2<sup>K+1</sup> to the balance. The new leading one at K+1 has
value -2<sup>K+1</sup>{{</html>}}, though, so adding a leading one preserves
the numeric value of the encoded number. This is one way to see why -1 is
encoded as all bits ones. Start with a single one and add leading ones.

When the sign bit is set, clear bits for to get away from zero.

## Strategy using `MOVN`

Using `MOVN` and `MOVK` gives a pretty good strategy for negative numbers.
Sometimes, the instruction sequence takes fewer than 64-bits!

When setting a register to a constant `N`, pick a 16-bit block in `N` that is
not all bits set, and use a `MOVN` to set that block, which also sets all bits
outside the block to one. If no such block exists, then `N` has all bits set,
and a single `MOVN` is enough. Otherwise, set other blocks of `N` with `MOVK`.

If there is even a single 16-bit block of ones in `N`, the initial `MOVN`
finishes setting two blocks or more in one go, beating out only using `MOVK`.
`MOVZ` covers positive numbers and `MOVN` covers the negatives.

This strategy is not limited to negative numbers, though. For example,
`0x7000_ffff_cafe_ffff` can be set efficiently with two instructions.

## Play with it!

On a browser with WASM support, there should be an interactive toy below that
implements the `MOVN` strategy. It only uses `MOVN` and `MOVK`, so
don't expect the shortest possible sequence for all inputs. (For one, it won't attempt to use
a [bitmask
immediate](https://dougallj.wordpress.com/2021/10/30/bit-twiddling-optimising-aarch64-logical-immediate-encoding-and-decoding/).)
It's made with [`capstone-rs`](https://github.com/capstone-rust/capstone-rs)
compiled through
[`wasm32-unknown-emscripten`](https://doc.rust-lang.org/stable/rustc/platform-support/wasm32-unknown-emscripten.html).
Please excuse the huge 3MB download size -- I made no effort minimizing it.

{{<html>}}
<input type="text" id="constantInput" value="0x7000_ffff_cafe_ffff"></input>
<br>
<pre id=disasmOutput style="height: 4lh;"></pre>
<script>
    Module = {
        noExitRuntime: true,
        onRuntimeInitialized() {
            // XXX: Synced with RUST
            const DISASM_TEXT_SIZE = 0x400

            // Smoke test inputs:
            // * the unsigned and signed representation of boundary values 
            // * 0
            // * -0x8000_0000_0000_0000, 0x8000_0000_0000_0000
            // * -1, 0xffff_ffff_ffff_ffff
            // * Just-out-of-range values

            constantInput.addEventListener("input", event => {
                disasmOutput.textContent = "Failed to parse input as a number"
                let input = event.target.value
                input = input.trim()
                if (!input) return

                // Support underscore separators
                input = input.replace(/_/g, "")

                // BigInt doesn't support negative hex values like -0x8,
                // so we hack it in here. Doesn't support 0b and 0o.
                let neg_hex = input.match(/^-0x\p{Hex_Digit}+$/u)
                if (neg_hex) input = input.substring(1)
                try {
                    n = BigInt(input)
                } catch (_) {
                    return // already shown error above.
                }
                if (neg_hex) n = -n

                if (n > 0xffff_ffff_ffff_ffffn || n < -0x8000_0000_0000_0000n) {
                    disasmOutput.textContent = "Number is too big to encode in 64 bits"
                    return
                }

                let disasm = Module._malloc(DISASM_TEXT_SIZE)
                Module.ccall("disasm", "number", ["number", "number"], [n, disasm])
                let disasm_text = Module.UTF8ToString(disasm)
                Module._free(disasm)
                disasmOutput.textContent = disasm_text
            })
            constantInput.dispatchEvent(new InputEvent("input"))
        }
    }
</script>
<script src="/arm64_movn.js" defer></script>
{{</html>}}
