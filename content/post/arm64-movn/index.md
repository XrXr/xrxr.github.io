+++
title = "ARM64's MOVN instruction is clever"
date = 2025-09-16T00:00:01-04:00
build.publishResources = false
+++

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

Adding [`MOVZ`](https://developer.arm.com/documentation/ddi0602/2025-06/Base-Instructions/MOVZ--Move-wide-with-zero-?lang=en) to the mix improves this strategy. Small numbers have many zeros
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

Leading ones are to negative numbers what leading zeros are to positive
numbers. [^1]

[^1]: It follows that [`typeof(n)::BITS - n.leading_zeros() - n.leading_ones() + 1`](https://doc.rust-lang.org/std/primitive.u64.html#method.leading_zeros) gives the
minimum number of bits it takes to encode `n`. One of the terms
is always zero.

## Strategy using `MOVN`

Using `MOVN` and `MOVK` gives a pretty good strategy for negative numbers.
Sometimes, the whole sequence takes fewer than 64 bits. That's shorter than
`movabs` from x86-64, which encodes all 64 bits of the constant into the
instruction.

Pick a 16-bit chunk in the constant that is not all bits set and use a `MOVN`
to set the chunk, filling all other chunks with ones as well. If no such chunk
exists, then all bits are set, and a single `MOVN` finishes the job. Otherwise,
set other chunks with `MOVK`.

If there is even a single 16-bit chunk of ones, the initial `MOVN` finishes
setting two chunks or more in one go, beating out the one-chunk-at-a-time
`MOVK`. `MOVZ` fills zeros and `MOVN` fills ones.

This strategy is not limited to negative numbers. For example, it sets
`0x7000_ffff_cafe_ffff` with just two instructions.

## Play with it!

On a browser with WASM support, there should be an interactive toy below that
implements the `MOVN` strategy. It only uses `MOVN` and `MOVK`, so don't expect
the shortest possible sequence for all inputs. (For one, it doesn't attempt to
use a [bitmask
immediate](https://dougallj.wordpress.com/2021/10/30/bit-twiddling-optimising-aarch64-logical-immediate-encoding-and-decoding/).)
It's made with [`capstone-rs`](https://github.com/capstone-rust/capstone-rs)
compiled through
[`wasm32-unknown-emscripten`](https://doc.rust-lang.org/stable/rustc/platform-support/wasm32-unknown-emscripten.html).
It assembles the machine code bytes only to run it through a disassembler as an
elaborate way of verification.

{{<html>}}
<input type="text" id="constantInput" value="0x7000_ffff_cafe_ffff"></input>
<br>
<pre id="disasmOutput"></pre>
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
{{</html>}}

{{<publish-resource "arm64_movn.wasm">}}
{{<page-bundle-js   "arm64_movn.js">}}
