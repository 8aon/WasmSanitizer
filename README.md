# instrumentation

Compile: `cargo build --bin test`

Use:  `./target/debug/test input.wasm output.wasm [func]`

Function list: [return value] [function name]

>- 11 null_reference
>- 12 implicit_signed_integer_truncation
>- 13 implicit_unsigned_integer_truncation
>- 14 float_cast_overflow
>- 15 non_return
>- 16 implicit_integer_sign_change
>- 17 shift
>- 18 unsigned_integer_overflow
>- 19 signed_integer_overflow
>- 20 float_divide_by_zero
>- 21 heap_canary
>- 22 stack_canary


## Changes 

### Why `unsigned_integer_overflow` is not part of `all`

The `unsigned_integer_overflow` pass is still available as a standalone mode, but `all` does not
run it, because it stops correct programs. In an evaluation on 53 benchmark runs it trapped in 4
programs (gnuchess, snappy, espeak, facedetection), each time on intended behavior. There are two
reasons, and neither can be fixed in the instrumentation:

1. **Unsigned wraparound is not an error in C.** The C standard defines unsigned arithmetic modulo
   2^N, so a sum or product that wraps around is legal, and programs rely on it, e.g. in hash
   functions, checksums and random number generators. This is why clang's
   `-fsanitize=unsigned-integer-overflow` is not part of `-fsanitize=undefined`: it is an opt-in
   check for finding suspicious code, not for running real programs. This pass does not just report
   a wraparound, it executes `unreachable` and terminates the program. Examples from the evaluation:
   - gnuchess `Initialize`: a table generator that keeps adding two values (`t = a + b`, starting
     from 1, 1) until the sum wraps around.
   - snappy `CompressFragment`: a hash function that multiplies by a constant and relies on the
     wraparound.
   - espeak `Translator::CalcPitches` and the PNG decoder of stb_image in facedetection.

2. **Wasm does not distinguish signed and unsigned arithmetic.** C has signed and unsigned types,
   but Wasm has a single `i32.add`/`i32.mul` (and `i64` variants) for both. The pass sees
   `local.get a; local.get b; i32.add; local.set c` and traps if the result, read as an unsigned
   number, is smaller than `a` (for `mul`, it compares with the 64-bit product). For signed code
   such as `x + y` with a negative `y`, the bit pattern of `y` is a huge unsigned number, so the
   addition "wraps around" as unsigned and the check fires, even though the signed result is
   correct. The binary contains no information to tell the two cases apart.

Note: an older version of the pass had an aliasing bug: for `x = x + y` it overwrote `x` before the
check and then compared the sum with itself, so the check could never fire for such statements. The
fix (keeping the result in a fresh local) made the check work as designed, which exposed these false
positives (e.g. gnuchess only started trapping after the fix).

Possible alternatives to leaving it out: keep it trapping and accept failing programs, or change it
to count/report violations without stopping the program.

### Why `implicit_integer_sign_change` is not part of `all`

The `implicit_integer_sign_change` pass is still available as a standalone mode, but `all` does not
run it, because it stops correct programs. In the same evaluation it trapped in 9 of the 53 runs
(blowfish (both runs), cjpeg, crc32, djpeg, espeak, facedetection, rijndael, snappy), each time on
correct code.

What the pass does: after `i32.load8_u; local.set y` (and the 16-bit and `i64` variants), it traps
if `y` would be negative when read as a *signed* value of the loaded width, i.e. for every byte
>= 128 (every 16-bit value >= 32768). It also inserts checks at the start of `__memcpy`, `memmove`
and `strncpy` that trap if the size argument is negative as a signed number, and at the start of
`dlmalloc` that trap if the size is <= 0 as a signed number (which includes the legal `malloc(0)`).

Why this cannot work on binaries:

1. **`load8_u` is simply how `unsigned char` is loaded.** The compiler uses `i32.load8_u` for every
   read of an `unsigned char`/`uint8_t` (and `load16_u` for `unsigned short`). Values >= 128 are
   completely normal for such types: any program that processes binary data (compression,
   encryption, checksums, image decoding) or non-ASCII text reads them all the time. A sign change
   would only be a problem if the value were later converted to a *signed* type, and whether that
   happens is not visible from the load.
2. **It is not undefined behavior.** clang's `-fsanitize=implicit-integer-sign-change` is part of
   the opt-in `-fsanitize=implicit-conversion` group, not of `-fsanitize=undefined`, and it works on
   the source-level types of each conversion. A Wasm binary contains neither the types nor the
   conversions, so the pass has to guess, and the guess is wrong for ordinary `unsigned char` code.
3. **It also instruments libc.** Unlike the other passes, it does not skip functions whose names
   start with `__` (that check is commented out), so it also fires inside libc. Examples from the
   evaluation: crc32 in `__uflow` (called by `getc` when reading the input file), blowfish in
   `__overflow` (called by `fputc`), espeak in `strlen` (on non-ASCII dictionary text).

Interaction with other passes: because the check depends on the byte values that are loaded, other
passes can make it fire or not by changing the memory layout. For example, in an earlier build,
bzip2 passed with this pass alone and with `stack_canary` alone, but trapped in this pass when both
were combined, because the stack canary moved data on the stack.

### Why `non_return` is not available

The `non_return` pass is not available at all (it is commented out of the CLI), because it produces
invalid Wasm modules and cannot be made to work on binaries.

What it was meant to do: detect functions with a return type that reach their end without a
`return` (in C, falling off the end of a non-`void` function and using the result).

Why it produces invalid modules: in the `wasm` library used here, a function's `code.body` always
ends with the function's final `end` instruction (required by the binary format). The pass checks
whether the *last* instruction is `return` or `unreachable`, which is never true because it is
always `end`, and then appends `unreachable` *after* the final `end` of every function that returns
a value. The result is rejected by every engine ("trailing code after function end").

Why it cannot be fixed by moving the `unreachable` before the final `end`: that would produce a
valid module, but break the program. Compilers return from functions by "falling through": the
result is left on the stack and the function simply reaches its `end`, without a `return`
instruction. In trisolv, 38 of the 39 functions that return a value end like this, so an
`unreachable` before `end` would trap on every normal return.

Why the check is not possible on binaries: Wasm validation requires a value of the right type on the
stack at the end of every function, so every valid function "returns something" on every path. When
clang sees a missing `return`, it either emits `unreachable` there itself, or returns some leftover
value. In the binary, a function that returns such a leftover value looks exactly like one that
returns a correctly computed value (e.g. `local.get N; end`). This bug can only be detected at the
source level, e.g. with clang's `-Wreturn-type` warning or `-fsanitize=return` (C++).

