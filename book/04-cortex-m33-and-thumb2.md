# Chapter 4 — The Cortex-M33 and Thumb-2

This chapter is a closer look at the processor that runs your rp-asm
code. Don't try to memorise everything here — treat it as a reference
to come back to. You'll absorb the details as you write programs.

## The register file

The Cortex-M33 has 16 general-purpose 32-bit registers. Most rp-asm
documentation refers to them by the names below.

| Name | Alias | Role |
| --- | --- | --- |
| `r0`–`r3` | — | Argument and return registers. **Caller-saved.** |
| `r4`–`r11` | — | General-purpose. **Callee-saved.** |
| `r12` | `ip` | Scratch / intra-procedure temporary. **Caller-saved.** |
| `r13` | `sp` | Stack pointer. |
| `r14` | `lr` | Link register (return address). |
| `r15` | `pc` | Program counter. |

"Caller-saved" means: if the caller cares about the value, the caller
saves it before the call. "Callee-saved" means: a called function must
preserve the value, so the caller can rely on it surviving the call.
We unpack this fully in chapter 8.

There is also a hidden register called `APSR` — the Application
Program Status Register — containing the four condition flags:

- **N** (negative): set if the result was negative.
- **Z** (zero): set if the result was zero.
- **C** (carry): set on unsigned overflow or by shifts.
- **V** (overflow): set on signed overflow.

You never name `APSR` directly in code; instructions that end in `s`
write to it, and conditional branches read from it.

## ARM, Thumb, and Thumb-2

ARM cores historically supported two instruction encodings:

- **ARM mode.** Every instruction is 32 bits. Flexible but bulky.
- **Thumb mode.** Every instruction is 16 bits. Smaller code, slightly
  fewer features.

Then ARM introduced **Thumb-2**, which mixes 16-bit and 32-bit Thumb
encodings in the same instruction stream. Each instruction is whichever
size it needs to be. This is the best of both worlds: dense code where
a 16-bit form exists, and the full power of 32-bit when it doesn't.

The Cortex-M33 **only** runs Thumb-2. There is no ARM mode at all on
M-profile cores. This is why every rp-asm function is preceded by:

```asm
    .thumb_func
    .global  my_function
my_function:
    ...
```

The `.thumb_func` directive tells the assembler "the symbol that
follows is Thumb code, so set bit 0 of its address". The bottom bit of
any function pointer on Cortex-M means "I am Thumb code"; if you ever
jump to an address with the bottom bit *clear*, the CPU will fault.

You don't need to think about this directly — `.thumb_func` does it for
you — but it explains a few small mysteries down the road.

## The instruction set, in broad strokes

You don't need to learn every instruction up front. The everyday
working set is small. Here's what you'll see constantly in rp-asm:

### Move and load constants

```asm
    movs    r0, #25         @ r0 = 25 (small immediate, < 256)
    mov     r1, r2          @ r1 = r2
    ldr     r0, =0x40014000 @ r0 = the 32-bit constant 0x40014000
    ldr     r0, =banner     @ r0 = address of label "banner"
```

`movs` with `#` takes a small immediate (up to 8 bits in most short
forms; up to a 12-bit modified immediate in 32-bit forms). When you
need a full 32-bit constant, write `ldr r0, =VALUE` — the assembler
will park the constant in a nearby pool and turn the instruction into
a PC-relative load.

### Arithmetic

```asm
    adds    r0, r1, r2      @ r0 = r1 + r2 (and update flags)
    subs    r0, #1          @ r0 = r0 - 1
    muls    r0, r1, r0      @ r0 = r1 * r0
    sdiv    r0, r1, r2      @ r0 = r1 / r2 (signed)
    udiv    r0, r1, r2      @ unsigned
```

### Bit operations

```asm
    ands    r0, r1          @ r0 &= r1
    orrs    r0, r1          @ r0 |= r1
    eors    r0, r1          @ r0 ^= r1
    bics    r0, r1          @ r0 &= ~r1
    lsls    r0, r1, #5      @ r0 = r1 << 5
    lsrs    r0, r1, #2      @ r0 = r1 >> 2 (logical, fills with 0)
    asrs    r0, r1, #2      @ r0 = r1 >> 2 (arithmetic, sign-extends)
```

### Memory access

```asm
    ldr     r0, [r1]        @ r0 = *(uint32_t*)r1
    ldr     r0, [r1, #4]    @ r0 = *(uint32_t*)(r1 + 4)
    str     r0, [r1]        @ *(uint32_t*)r1 = r0
    ldrb    r0, [r1]        @ r0 = *(uint8_t*)r1 (zero-extended)
    ldrh    r0, [r1]        @ r0 = *(uint16_t*)r1
    strb / strh             @ store byte / halfword
```

The `[...]` syntax is "the memory address inside these brackets". You
can also write `[r1, r2]` (register-indexed), `[r1], #4` (post-increment
r1 after the access), and `[r1, #4]!` (pre-increment, write back).
You'll see them all in real rp-asm code.

### Compare and branch

```asm
    cmp     r0, r1          @ compute r0 - r1, set flags, discard result
    beq     .Lequal         @ branch if Z=1 (equal)
    bne     .Lloop          @ branch if Z=0
    blt     .Lneg           @ branch if signed less-than
    cbz     r0, .Lzero      @ compare-and-branch-if-zero (r0 only)
    cbnz    r0, .Lnonzero
    b       .Ldone          @ unconditional branch
    bl      function        @ branch with link (function call)
    bx      lr              @ branch to register (function return)
```

`bl` saves the address of the next instruction into `lr` so that the
called function can return to it. `bx lr` returns by jumping to `lr`.
The `x` in `bx` means "interworking" — the CPU looks at bit 0 to know
whether the destination is Thumb. Since all our code is Thumb, bit 0
is always 1, and `bx lr` is what a function-return looks like.

### Pushing and popping

```asm
    push    {r4, r5, lr}    @ predecrement sp, store r4, r5, lr
    pop     {r4, r5, pc}    @ load into r4, r5, pc; postincrement sp
```

`push` and `pop` are how you save callee-saved registers across calls,
and how you save `lr` if you'll do a `bl` inside your own function.
Popping into `pc` is the standard return idiom for functions that
push `lr`.

## The memory map

On the RP2350, the 32-bit address space is divided into regions. The
ones you'll care about in this book are:

| Address range | What lives here |
| --- | --- |
| `0x00000000`–`0x00007fff` | Bootrom (32 KB, read-only) |
| `0x10000000`–`0x103fffff` | QSPI flash, mapped via XIP cache (4 MB on Pico 2) |
| `0x20000000`–`0x20081fff` | SRAM (520 KB) |
| `0x40000000`–`0x4fffffff` | APB peripherals — UART, GPIO, I2C, etc. |
| `0xd0000000`–`0xd000ffff` | SIO (single-cycle I/O, includes GPIO out) |
| `0xe0000000`–`0xe00fffff` | Cortex-M33 system control (NVIC, SysTick) |

The two regions you'll talk to most are SRAM (your stack, your data,
sometimes your code during testing) and the peripheral region. Every
hardware peripheral on the chip has a base address; you write to a
specific offset from that base to control it. We meet this concept
formally in chapter 9 as **memory-mapped I/O**.

## Atomic register aliases — the RP2 trick

Most peripheral registers on most chips require read-modify-write to
change one bit: read the register, OR in a bit, write it back. That
takes three instructions and is unsafe against interrupts.

The RP2350 (and the RP2040 before it) cleverly map every peripheral
register four times:

| Offset | Effect |
| --- | --- |
| `+0x0000` | Normal read/write |
| `+0x1000` | Atomic XOR (writing 1 toggles that bit) |
| `+0x2000` | Atomic SET (writing 1 sets that bit) |
| `+0x3000` | Atomic CLR (writing 1 clears that bit) |

So to toggle GP25's output bit you do not need a read-modify-write.
You just store a 1 in the right place at the right alias and the
hardware does the bit-flip in two cycles.

rp-asm uses this aggressively. You'll see, for example:

```asm
    ldr     r0, =SIO_BASE + SIO_GPIO_OUT_XOR_OFFS
    movs    r1, #(1 << 25)
    str     r1, [r0]            @ toggle GP25 atomically
```

That single `str` is the entire LED toggle. Two cycles, no scratch
register, no race with an ISR. This idiom is *the* signature move of
RP2 assembly and one of the prettiest tricks on the chip.

## What state is the CPU in at boot?

When the bootrom hands off to your reset handler, you can rely on:

- The CPU is in Thumb mode (it's always in Thumb mode on M33).
- The MSP (main stack pointer) is loaded from the first word of your
  vector table.
- The PC is loaded from the second word of your vector table.
- Interrupts are disabled at the NVIC level (no peripheral has yet
  been enabled to fire).
- The XOSC is running at 12 MHz and is the source for `clk_sys`.

You inherit a clean machine. From there, what happens next is whatever
you write — which is the whole point of this book.

## What's next

You now have a working vocabulary of registers, instructions, and the
RP2350 memory map. The next chapter gets the toolchain installed so
we can turn assembly source into a runnable `.uf2` file.
