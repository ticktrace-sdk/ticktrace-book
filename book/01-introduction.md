# Chapter 1: Introduction

This book is a working introduction to ARM Thumb-2 assembly
programming on the Raspberry Pi RP2350 microcontroller. It uses
**rp-asm**, a pure-assembly SDK for that chip, as its running
codebase.

**Scope:** 14 chapters and two appendices, covering the CPU
architecture, the toolchain, the calling convention, the GPIO, UART,
and timer/interrupt subsystems, and the scheduling and multicore
primitives rp-asm provides.

**Outcome:** the reader will be able to build, flash, and modify
cycle-counted bare-metal firmware on a Raspberry Pi Pico 2 without
invoking a C compiler at any step in the pipeline.


## What this book is

It is a beginner's introduction to **assembly language**, taught through
a concrete project: writing firmware for the Raspberry Pi Pico 2 using
the rp-asm SDK. We will spend a chapter or two on theory, what a CPU is,
what an instruction is, what the RP2350 chip looks like, and then we
spend the rest of the book writing real code that runs on real hardware.

## What this book is not

It is **not** a reference manual. The RP2350 datasheet is 1300 pages and
freely available; we will quote the bits we need and trust you to look
up the rest when curious. It is also not a tour of every instruction in
the ARM Thumb-2 set. We teach what you actually need to read and write
rp-asm code, and we point you at deeper references when it's time.

It is also not a book about computer architecture in the abstract. There
are wonderful books about pipelines, branch predictors, cache coherency,
and out-of-order execution, etc. I couldn't teach you these even if I wanted to as I myself am not an expert. 
The Cortex-M33 has none of those things in any interesting form  it is a simple, in-order processor, and that is
exactly why it is a good teaching target.

## Why I built rp-asm

I write firmware for motor control. In that world, the latency of an
ISR matters down to the cycle, the jitter on a PWM edge can wreck a
control loop, and "the compiler probably did the right thing" is not
an acceptable answer. Deterministic real-time work is hard when you
don't know what the compiler does.

Before rp-asm I lived in two languages on this chip.

**C, usually with the pico-sdk.** Powerful, and the SDK takes care of
bring-up, clocks, USB, every block you'd otherwise spend a month on.
That safety is worth a lot. But the abstractions stack up. The build
system grows tentacles. You start asking the compiler not to reorder
this load, not to inline that function, to please honour your timing
budget, and at some point you realise you've been arguing with a tool
that doesn't share your goals. Power, yes. Predictability, only if you
fight for it.

**TinyGo.** A small miracle: Go on a microcontroller, with a syntax I
genuinely enjoy. For prototyping it's hard to beat. But the runtime,
the GC, the reflection metadata, they show up in the binary, and for
the tight inner loops of a motor controller the indirection is exactly
the thing you can't afford.

Both languages have their place. Neither was the right tool for the
work I actually wanted to do, which is *write the exact sequence of
instructions that hits the wire at the exact cycle I expect*.

rp-asm is what that looks like when you take it seriously. Pure
Thumb-2 assembly. **No compiler between you and the silicon**. Every
register, every peripheral, every cycle accounted for in source you
can read end-to-end. The motto on the front of the README, "every
cycle matters", isn't a slogan, it's the reason the project exists.

This book is the introduction I wish I'd had when I was learning to
think in cycles instead of statements.

## Why a microcontroller is a great place to learn assembly

When you learn a high-level language, you typically start with a "hello,
world" that hides almost everything: a `main` function appears in a
running process, a `print` function appears in a standard library, and
the words appear on a terminal. The whole stack is opaque.

When you learn assembly on a desktop OS, you get *almost* the same
problem in reverse: you can write `mov` and `add` instructions all day,
but the moment you want to actually *do* something, read a file, draw
a pixel, sleep for a second, you have to call into an operating system
that hides almost everything below it.

A microcontroller has no operating system. The chip boots, jumps to your
code, and your code runs forever. There is no kernel to ask for
permission, no scheduler to preempt you, no syscall layer. If you want to
blink an LED, you write to a hardware register. If you want to print to
a terminal, you push bytes into a UART. Every layer is visible. Every
layer is yours.

rp-asm makes this concrete by giving you, in pure assembly:

- A startup file that boots the chip
- Drivers for every peripheral on the RP2350
- Working examples
- A test harness so you can verify changes without burning an LED

You will figure out exactly what your chip does. There is no part of the system that is somebody
else's secret.

## What you'll need

**Hardware:**

- A Raspberry Pi Pico 2 (or Pico 2 W). About US$5.
- A USB cable that fits the Pico 2 (USB-A or USB-C depending on the cable
  side, and USB-C to the Pico 2 itself for the Pico 2 board).
- A computer running Linux. macOS works too with minor toolchain
  differences; Windows works under WSL.

**Software:**

- The GNU ARM embedded toolchain (`binutils-arm-none-eabi`)
- Python 3 (for the rp-asm test harness)
- A text editor

Chapter 5 walks you through installing all of this. If you don't have a
Pico 2 yet you can still follow the book, the rp-asm test harness lets
you run programs in emulation  but you'll miss the satisfaction of the
blinking LED.

## How to read

Read in order. The chapters are short on purpose, and each one assumes
the previous one. When you hit a code example, type it out yourself
rather than copy-pasting; the muscle memory is part of the point.

When you finish chapter 6 you will have a working program. From there
you can either keep reading linearly or skip ahead to whichever
peripheral chapter interests you, they are mostly independent.

Onward.

<!-- nav-footer -->

---

[Table of contents](README.md) · [Chapter 2: What is assembly language? →](02-what-is-assembly.md)
