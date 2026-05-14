# Learning Assembly with rp-asm

A book for total beginners who want to understand what a CPU really does
and write their own bare-metal firmware for the Raspberry Pi Pico 2.

> "Every cycle matters." — the rp-asm motto.

## Who this book is for

You should be comfortable with **some** programming language — C, Python,
Rust, JavaScript, anything where you've written a `for` loop and called a
function. You do **not** need any prior experience with:

- Assembly language
- ARM processors
- Embedded systems
- Microcontrollers, soldering, electronics
- The Raspberry Pi Pico

If you can install a package on Linux and you own (or are willing to buy) a
**Raspberry Pi Pico 2** (about US$5), you have everything you need.

## What you will learn

By the end of the book you will:

1. Understand what a CPU register is, what memory-mapped I/O is, and what an
   instruction set is.
2. Read and write ARM Thumb-2 assembly fluently enough to follow the rp-asm
   source tree.
3. Build, flash, and debug your own assembly programs on real Pico 2 hardware.
4. Write programs that talk to the outside world over UART, blink LEDs,
   handle timer interrupts, and coordinate with the rp-asm driver library.

## Why assembly? Why now?

Most of the world's software is written in high-level languages, and that
is the right default. But assembly is still worth learning because:

- **It is the truth.** Every C, Rust, or Python program eventually becomes
  the instructions you are about to learn. Reading assembly turns "magic"
  into mechanism.
- **Microcontrollers are small.** When you have 520 KB of RAM and need to
  drive USB, an LED, and a timer, knowing what each cycle costs stops being
  academic.
- **It is fun.** There is a particular pleasure in seeing a 728-byte binary
  bring up a clock tree, configure a UART, and start blinking an LED — and
  understanding every byte.

rp-asm is, to our knowledge, the only fully-featured SDK for the RP2350
written entirely in assembly. It is small, complete, and (because we have
no C compiler hiding details) every line is something you can read.

## How to read this book

Chapters build on each other. The first half of the book — chapters 1–6 —
is the foundation. After chapter 6 you will have a blinking LED and a
"hello, world" banner on a serial terminal, written in your own assembly.
The second half — chapters 7–12 — explains the rp-asm idioms in depth and
walks you through GPIO, UART, and interrupts.

Each chapter is short. Run the examples as you go.

## Table of contents

| # | Chapter | Has figures? |
| --- | --- | --- |
| 1 | [Introduction](01-introduction.md) | — |
| 2 | [What is assembly language?](02-what-is-assembly.md) | register file, fetch/decode cycle |
| 3 | [The RP2 family](03-the-rp2-family.md) | chip block diagram, Pico 2 pinout, boot flow |
| 4 | [The ARM Cortex-M33 and Thumb-2](04-cortex-m33-and-thumb2.md) | register file, memory map, atomic aliases |
| 5 | [Setting up rp-asm](05-setting-up-rp-asm.md) | build pipeline |
| 6 | [Your first program: blinky](06-your-first-program.md) | boot sequence, image layout |
| 7 | [Assembler syntax and instructions](07-assembler-syntax.md) | section linking |
| 8 | [Functions and the calling convention](08-functions-and-calling-convention.md) | call sequence, stack frame |
| 9 | [GPIO and memory-mapped I/O](09-gpio-and-memory-mapped-io.md) | pin block, CTRL register |
| 10 | [UART: talking to the host](10-uart.md) | UART frame, init sequence |
| 11 | [Timers and interrupts](11-timers-and-interrupts.md) | vector table, IRQ entry |
| 12 | [Scheduling](12-scheduling.md) | scheduler dispatch |
| 13 | [Multicore](13-multicore.md) | launch handshake |
| 14 | [Where to go next](14-where-to-go-next.md) | — |
|   | [Appendix A — Glossary](A-glossary.md) | — |
|   | [Appendix B — Cheat sheet](B-cheat-sheet.md) | — |

Every chapter from 2 onward ends with exercises. Skim them even if
you don't work them — they're often the questions a reader actually
has at that point.

## Conventions used in this book

- Code blocks labelled `asm` are ARM Thumb-2 assembly in GNU syntax — the
  same syntax rp-asm uses.
- Code blocks labelled `console` show shell commands. Lines starting with
  `$` are typed; everything else is output.
- `file.S:42` references mean "line 42 of file.S in the rp-asm tree".
- Hexadecimal numbers are written with a `0x` prefix; binary with `0b`.
- "The datasheet" always means the *RP2350 Datasheet* from Raspberry Pi.

## On the figures

Diagrams come in three flavours, all rendered natively by GitHub:

- **SVG files** in [`book/figures/`](figures/) for hardware layouts,
  register bit-fields, stack frames, and memory maps. Open them in
  a browser to zoom or copy.
- **Mermaid diagrams** inline for flowcharts and sequences.
- Occasional **ASCII art** in code blocks for tiny inline visuals.

If you're reading on a renderer that doesn't display SVG or Mermaid
(e.g. plain text), the surrounding prose still stands on its own.

Let's begin.
