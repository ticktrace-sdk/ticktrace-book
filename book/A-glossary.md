# Appendix A: Glossary

Every term used in this book, with a one-paragraph definition and a
pointer to the chapter that introduces it.

---

**AAPCS.** ARM Architecture Procedure Call Standard.
The set of rules every function in rp-asm follows so that functions
can call each other safely. Arguments go in `r0`–`r3`; return value in
`r0` (low) and `r1` (high); `r4`–`r11` are callee-saved; the stack
stays 8-byte aligned at every call boundary. See
[chapter 8](08-functions-and-calling-convention.md).

**ADC.** Analog-to-Digital Converter.
A peripheral that samples a voltage on a pin and produces a digital
number. The RP2350 has an 8-channel, 12-bit ADC plus an on-die
temperature sensor. See `src/adc.S` and `docs/adc.md`.

**APB.** Advanced Peripheral Bus.
The on-chip bus that connects the CPU to most peripherals (UART, I²C,
SPI, GPIO config, etc.). Lives at `0x40000000`–`0x4FFFFFFF` on the
RP2350. See [chapter 4](04-cortex-m33-and-thumb2.md).

**APSR.** Application Program Status Register.
The processor's flag register, holding N (negative), Z (zero), C
(carry), and V (overflow). Updated by instructions with the `s`
suffix (`adds`, `subs`, `lsls`, …); read by conditional branches.
See [chapter 4](04-cortex-m33-and-thumb2.md).

**Assembler.**
The tool that turns assembly source code (text mnemonics) into machine
code (binary opcodes). In rp-asm this is `arm-none-eabi-as` from
binutils. See [chapter 5](05-setting-up-rp-asm.md).

**Atomic alias.**
A second address that points at the same hardware register but applies
a SET, CLR, or XOR operation atomically. On RP2350 APB peripherals
the aliases live at base + 0x1000 (XOR), + 0x2000 (SET), + 0x3000
(CLR). One STR, two cycles, no race. See
[chapter 4](04-cortex-m33-and-thumb2.md) and
[chapter 9](09-gpio-and-memory-mapped-io.md).

**Baud rate.**
The signalling rate on a serial line, in bits per second. rp-asm
defaults to 115200 baud on UART0, about 8.7 µs per bit, or ~11.5 KB/s
of one-way throughput. See [chapter 10](10-uart.md).

**bl.** Branch with Link.
A function-call instruction. Saves the return address in `lr`, then
jumps to the target. The standard return is `bx lr` or `pop {…, pc}`.
See [chapter 2](02-what-is-assembly.md).

**.bss.**
The section for zero-initialised RAM data. Lives in SRAM; the reset
handler fills it with zero before `main` runs. See
[chapter 7](07-assembler-syntax.md).

**Bootrom.**
A 32 KB read-only program baked into the RP2350 silicon. Runs first on
power-on, checks BOOTSEL, then either presents USB mass storage or
launches your flash image. See [chapter 3](03-the-rp2-family.md).

**BOOTSEL.**
A button on the Pico 2. Hold it while plugging in USB to make the
bootrom enter mass-storage mode for drag-and-drop UF2 flashing. See
[chapter 5](05-setting-up-rp-asm.md).

**Callee-saved.**
A register that a called function must preserve. On AAPCS: `r4`–`r11`.
If a function wants to use one, it must push it on entry and pop it
on return. See [chapter 8](08-functions-and-calling-convention.md).

**Caller-saved.**
A register that a called function is free to clobber. The caller must
save it before the call if it cares about the value. On AAPCS:
`r0`–`r3`, `r12`. See [chapter 8](08-functions-and-calling-convention.md).

**CDC.** Communications Device Class.
A USB standard for serial-like devices. rp-asm's USB driver implements
CDC-ACM, which makes a Pico 2 appear as `/dev/ttyACM0` on a Linux
host. See [chapter 10](10-uart.md) and `docs/usb.md`.

**Cooperative scheduling.**
A scheduler where each task runs until it voluntarily yields. Simple
to reason about, but a misbehaving task can hold the CPU forever.
The superloop is the simplest case. See
[chapter 12](12-scheduling.md).

**Core 0 / Core 1.**
The two Cortex-M33 cores on the RP2350. Core 0 boots first and runs
your `main`. Core 1 sits in the bootrom until `multicore_launch_core1`
sends the 6-word handshake. See [chapter 13](13-multicore.md).

**Cortex-M33.**
The ARMv8-M Mainline CPU core inside the RP2350. Runs Thumb-2; has a
full register file with optional FPU and DSP extensions. See
[chapter 4](04-cortex-m33-and-thumb2.md).

**CPACR.** Coprocessor Access Control Register.
A CPU control register that enables or disables coprocessors,
including the RCP (Random Canary Protection) on RP2350. The reset
handler writes it early.

**Critical section.**
A run of instructions during which interrupts are disabled (`cpsid i`)
to protect a shared data structure from concurrent ISR access.
Re-enable with `cpsie i`. Use sparingly, disabled interrupts mean
missed events. See [chapter 11](11-timers-and-interrupts.md).

**DMA.** Direct Memory Access.
A peripheral that copies data from one address to another without CPU
involvement. The RP2350 has 16 DMA channels. See `src/dma.S` and
`docs/dma.md`.

**.equ.**
A GNU assembler directive that defines a compile-time constant.
Equivalent to C's `#define`. See [chapter 7](07-assembler-syntax.md).

**EXC_RETURN.**
A special value the hardware loads into `lr` on interrupt entry. When
the ISR returns by branching to `lr` (typically via `pop {…, pc}`),
the hardware sees the magic pattern and unwinds the saved exception
frame. See [chapter 11](11-timers-and-interrupts.md).

**FIFO.** First-In, First-Out queue.
A small hardware buffer inside a peripheral (UART, SIO, etc.) that
smooths over short bursts of activity. The PL011 UART has a 32-byte
TX FIFO. See [chapter 10](10-uart.md).

**Flash.**
The non-volatile memory that stores your program. The Pico 2 has 4 MB
of external QSPI flash, mapped into the address space at
`0x10000000`–`0x103FFFFF` via the XIP cache. See
[chapter 4](04-cortex-m33-and-thumb2.md).

**FUNCSEL.**
The 5-bit field in `IO_BANK0.GPIO[n].CTRL` that selects which
peripheral drives a pin. 0 = XIP, 2 = UART, 5 = SIO, 6 = PIO0, etc.
See [chapter 9](09-gpio-and-memory-mapped-io.md).

**GPIO.** General-Purpose Input/Output.
A pin that can be configured to read a digital voltage or drive one.
The RP2350 has 48 user GPIOs (GP0..GP47). See
[chapter 9](09-gpio-and-memory-mapped-io.md).

**I²C.** Inter-Integrated Circuit.
A two-wire serial bus for talking to sensors and EEPROMs. The RP2350
has two DesignWare I²C controllers. See `src/i2c.S` and `docs/i2c.md`.

**IMAGE_DEF.**
A signature block at a fixed offset in your binary that tells the
bootrom how to launch the image. rp-asm emits one in `src/startup.S`.
See [chapter 3](03-the-rp2-family.md) and `docs/boot.md`.

**Interpolator.**
A small hardware block (two per core, INTERP0 and INTERP1) that
computes `((accum >> shift) & mask) + base` in a single cycle. Useful
for table lookup and fixed-point arithmetic in hot loops. See
[chapter 13](13-multicore.md) and `src/interp.S`.

**Interrupt / IRQ.**
A hardware-triggered jump from regular code into an ISR. Handled by
the NVIC; routed through the vector table. See
[chapter 11](11-timers-and-interrupts.md).

**ISO bit.**
A bit in each `PADS_BANK0.GPIO[n]` register that, when set, isolates
the pad from the SIO drive signal. All RP2350 pads reset with ISO=1;
every GPIO driver must clear it before the pin can drive a level.
See [chapter 9](09-gpio-and-memory-mapped-io.md).

**ISR.** Interrupt Service Routine.
A function the hardware jumps to when an interrupt fires. AAPCS still
applies to callee-saved registers; the hardware itself saves
caller-saved ones. See [chapter 11](11-timers-and-interrupts.md).

**ldr.** Load Register.
The Thumb-2 instruction that loads a 32-bit value from memory into a
register. Has many addressing forms; `ldr Rd, =VALUE` is a pseudo-
instruction the assembler turns into a PC-relative load from a literal
pool. See [chapter 4](04-cortex-m33-and-thumb2.md).

**Linker.**
The tool that combines object files into a final image, resolving
symbol references and laying out sections. In rp-asm this is
`arm-none-eabi-ld` with `link/flash.ld` (or `link/sram.ld`).
See [chapter 5](05-setting-up-rp-asm.md).

**Literal pool.**
A block of 4-byte constants the assembler emits at the end of (or
inside) a function, so PC-relative `ldr` instructions can reach them.
Usually invisible to you, the assembler manages it. See
[chapter 7](07-assembler-syntax.md).

**lr.** Link Register (r14).
Holds the return address after a `bl`. Functions that make further
`bl` calls must save it on entry and restore it before returning.
See [chapter 4](04-cortex-m33-and-thumb2.md).

**MMIO.** Memory-Mapped I/O.
The technique of controlling peripherals by reading and writing
special memory addresses. The basis of every rp-asm driver. See
[chapter 9](09-gpio-and-memory-mapped-io.md).

**NVIC.** Nested Vectored Interrupt Controller.
The CPU's interrupt controller. Has per-IRQ enable, pending, active,
and priority bits. Lives at `0xE000E100`. See
[chapter 11](11-timers-and-interrupts.md).

**Multicore.**
The RP2350 has two Cortex-M33 cores that share SRAM and peripherals
but have separate register files, stacks, NVICs, and vector tables.
See [chapter 13](13-multicore.md).

**OD bit.**
A bit in each `PADS_BANK0.GPIO[n]` register that disables the pad's
output driver. Set at reset alongside ISO; both must be cleared
before the pin can drive. See
[chapter 9](09-gpio-and-memory-mapped-io.md).

**Opcode.**
The actual bit pattern the CPU executes. Mnemonics like `movs` map to
opcodes through an assembler table lookup. See
[chapter 2](02-what-is-assembly.md).

**PADS_BANK0.**
The peripheral block that controls electrical settings of each user
GPIO pad, drive strength, pull-up/down, isolation, output disable,
input enable. Base address `0x40038000`. See
[chapter 9](09-gpio-and-memory-mapped-io.md).

**pc.** Program Counter (r15).
The address of the next instruction to execute. Writing to `pc` (via
`bx`, `pop`, `mov`, etc.) is how branches happen. See
[chapter 4](04-cortex-m33-and-thumb2.md).

**Preemptive scheduling.**
A scheduler where the runtime can interrupt a running task and switch
to another at any instruction. rp-asm's scheduler uses
NVIC-priority-based preemption: a higher-priority task preempts a
lower-priority one; same-priority tasks run to completion. See
[chapter 12](12-scheduling.md).

**Peripheral.**
A hardware block on the chip outside the CPU core, UART, I²C, SPI,
GPIO, timer, USB, DMA, ADC, etc. Each has a base address and a
register map. See [chapter 3](03-the-rp2-family.md).

**PIO.** Programmable I/O.
The RP2 family's unique state-machine engine that can be programmed to
bit-bang custom protocols. See `src/pio.S` and `docs/pio.md`.

**PL011.**
ARM's standard UART controller IP block. The RP2350's UART0 and UART1
are PL011 instances. See [chapter 10](10-uart.md) and `docs/uart.md`.

**PLL.** Phase-Locked Loop.
A clock multiplier. rp-asm uses PLL_SYS to turn the 12 MHz crystal
into 150 MHz `clk_sys`, and PLL_USB to produce 48 MHz `clk_usb`. See
`src/pll.S` and `docs/clocks.md`.

**Push / pop.**
The Thumb-2 instructions that save and restore register lists on the
stack. Always paired. See [chapter 8](08-functions-and-calling-convention.md).

**QSPI.** Quad SPI.
A four-wire variant of SPI used for the external flash on the Pico 2.
The chip's XIP cache fetches code from QSPI at run-time. See
`src/qmi.S`.

**RTOS.** Real-Time Operating System.
A piece of software providing a fully-featured scheduler plus
synchronisation primitives, timers, message queues, and dynamic task
creation. FreeRTOS, Zephyr, ThreadX, ChibiOS are common examples.
rp-asm's scheduler is deliberately *not* an RTOS, it covers a
narrower regime with much less code. See
[chapter 12](12-scheduling.md).

**RP2040.**
Raspberry Pi's first chip (2021): dual Cortex-M0+, 264 KB SRAM,
30 GPIOs. Used in the original Pico. See
[chapter 3](03-the-rp2-family.md).

**RP2350.**
Raspberry Pi's second chip (2024): dual Cortex-M33, 520 KB SRAM,
48 GPIOs. Target of rp-asm. See [chapter 3](03-the-rp2-family.md).

**.rodata.**
The section for read-only data, string literals, constant lookup
tables, etc. Lives in flash on hardware builds. See
[chapter 7](07-assembler-syntax.md).

**SIO FIFO.**
A 4-deep inter-core word mailbox in the SIO block, at
`SIO_BASE + 0x050..0x058`. Used for the `multicore_launch_core1`
handshake and for general-purpose core-to-core notifications. See
[chapter 13](13-multicore.md).

**SIO.** Single-cycle I/O.
A peripheral block attached directly to the CPU bus (no APB hop), for
fast access. Houses GPIO drive bits, hardware spinlocks, the inter-core
FIFO, and the interpolator units. Base address `0xD0000000`. See
[chapter 9](09-gpio-and-memory-mapped-io.md).

**sp.** Stack Pointer (r13).
Points at the current top of the stack. Decremented by `push`,
incremented by `pop`. Must be 8-byte aligned at every call boundary.
See [chapter 8](08-functions-and-calling-convention.md).

**Spinlock.**
A busy-wait lock implemented in hardware: read the address to attempt
acquisition, write to release. The RP2350 has 32 of them at
`SIO_BASE + 0x100 + n*4`. Use briefly, while one core holds the
lock, the other spins. See [chapter 13](13-multicore.md).

**SPI.** Serial Peripheral Interface.
A four-wire full-duplex serial bus. The RP2350 has two PL022 SPI
controllers. See `src/spi.S` and `docs/spi.md`.

**SPSC.** Single-Producer, Single-Consumer queue.
A lock-free byte ring buffer in `src/spsc.S`. Used to pass data from
ISRs (producer) to soft tasks (consumer) without needing locks,
because run-to-completion scheduling and the SPSC invariant together
remove the race. See [chapter 12](12-scheduling.md).

**SRAM.** Static RAM.
Volatile memory. 520 KB on the RP2350, mapped at `0x20000000`. Holds
the stack, `.bss`, `.data`. See [chapter 4](04-cortex-m33-and-thumb2.md).

**Superloop.**
The simplest scheduler: a single forever-loop in `main` that calls
each job in turn. Works when jobs are short and bounded. See
[chapter 6](06-your-first-program.md) and
[chapter 12](12-scheduling.md).

**Task (rp-asm).**
A function installed as an NVIC IRQ handler via `task_create`. Posted
with `task_post`; runs at the handler's NVIC priority; returns when
done. All tasks share the main stack. See
[chapter 12](12-scheduling.md).

**.text.**
The section for executable code. rp-asm puts each function in its
own subsection (`.text.foo`) so the linker can drop unused ones.
See [chapter 7](07-assembler-syntax.md).

**Thumb / Thumb-2.**
ARM's 16-bit-instruction encoding (Thumb) and its mixed 16/32-bit
extension (Thumb-2). Cortex-M cores run only Thumb-2. See
[chapter 4](04-cortex-m33-and-thumb2.md).

**.thumb_func.**
A GNU assembler directive that marks the next symbol as a Thumb
function, so its address has bit 0 set. Required on every function
that will be called via `bl`/`bx`. See
[chapter 7](07-assembler-syntax.md).

**UART.** Universal Asynchronous Receiver/Transmitter.
A serial port. The Pico 2 has two PL011 UARTs. See
[chapter 10](10-uart.md).

**UF2.** USB Flashing Format.
A simple block-based file format Raspberry Pi's bootrom understands.
rp-asm builds `*.uf2` files via the in-tree `rpasm` CLI
(`tools/bin/rpasm uf2 pack`) or Studio's GUI. See
[chapter 5](05-setting-up-rp-asm.md) and
[Appendix E](E-studio.md).

**Vector table.**
An array of 32-bit function pointers, one per CPU exception and one
per external IRQ. Lives at the start of your image. The `VTOR`
register tells the CPU where it is. See
[chapter 11](11-timers-and-interrupts.md).

**VTOR.** Vector Table Offset Register.
A CPU control register at `0xE000ED08` that points at the active
vector table. rp-asm's `_reset` writes it.

**wfi.** Wait For Interrupt.
A Thumb-2 instruction that halts the CPU until an enabled interrupt
fires. Saves power. See [chapter 11](11-timers-and-interrupts.md).

**XIP.** eXecute In Place.
A mode where the CPU fetches instructions directly from external
flash via a hardware cache, rather than copying everything to RAM
first. The QSPI flash on the Pico 2 is XIP. See `docs/boot.md`.

**XOSC.** Crystal Oscillator.
The 12 MHz crystal on the Pico 2 board, fed to the chip's clock
generator. Source for the PLLs. See `src/xosc.S` and `docs/clocks.md`.

<!-- nav-footer -->

---

[← Chapter 14: Where to go next](14-where-to-go-next.md) · [Table of contents](README.md) · [Appendix B: Cheat sheet →](B-cheat-sheet.md)
