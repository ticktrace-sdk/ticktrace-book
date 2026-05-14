# Chapter 14 — Where to go next

You can now read and write rp-asm. You understand registers,
instructions, the calling convention, memory-mapped I/O, GPIO, UART,
interrupts, scheduling, and multicore. That's enough vocabulary to
navigate the entire SDK. The rest is *what specifically* each
peripheral does — i.e., the datasheet.

This closing chapter is a map of the territory we *didn't* cover, with
pointers for where to look when you want to.

## More peripherals in `src/`

The drivers we touched explicitly were `gpio`, `uart`, `timer`, and
`nvic`. Here's the rest of the `src/` tree, with one line each on
what it's for and where to start reading:

- **`xosc.S`, `pll.S`, `clocks.S`, `tick.S`** — the clock-tree
  bring-up. `docs/clocks.md` walks through the math. Read these once
  to understand how `clk_sys` becomes 150 MHz.
- **`dma.S`** — the DMA controller. 16 channels that copy memory or
  drive peripherals without CPU involvement. `docs/dma.md`. Try
  `examples/dma_memcpy_demo.S`.
- **`pwm.S`** — pulse-width modulation. 12 slices, useful for fading
  LEDs, driving servos, generating audio. `docs/pwm.md`, plus
  `examples/pwm_fade_demo.S` and `examples/pwm_servo_demo.S`.
- **`i2c.S`** — DesignWare I2C controller. Master and slave modes.
  `docs/i2c.md`, `examples/i2c_eeprom_demo.S`.
- **`spi.S`** — PL022 SPI controller. Full-duplex, DMA-friendly.
  `docs/spi.md`.
- **`usb.S`** — the USB device controller. Implements a CDC-ACM
  endpoint so the Pico shows up as `/dev/ttyACM0`. Big file; reading
  it is a real expedition. `docs/usb.md`.
- **`adc.S`, `trng.S`, `sha256.S`** — analog input, true random
  numbers, hardware SHA-256. Small, focused drivers. The
  `examples/data_usb_demo.S` exercises all three.
- **`pio.S`** — the **Programmable I/O** state machines. Unique to
  this chip family: a tiny CPU you can program to bit-bang custom
  protocols. `docs/pio.md`, `examples/pio_blink_demo.S`.
- **`multicore.S`, `spinlock.S`, `interp.S`** — covered in
  [chapter 13](13-multicore.md). The drivers themselves are short
  and worth reading after the chapter to see the bit-level detail.
- **`watchdog.S`, `powman.S`, `systick.S`** — chip-level housekeeping.
- **`qmi.S`, `otp.S`, `bootrom.S`** — the QSPI flash interface, the
  one-time-programmable fuses, and bootrom services like
  "reset to BOOTSEL".
- **`sched.S`, `spsc.S`, `sched_stats.S`** — covered in
  [chapter 12](12-scheduling.md). Re-read the source once you've
  worked through the chapter; it's a small, complete data-structure
  showcase in pure asm.
- **`trace.S`** — CoreSight DWT and ITM, for cycle counting and
  on-chip `printf`-style debugging.

For each driver: read the `docs/` file, then skim the matching `src/`
file. The drivers are well-commented and short by software-engineering
standards (most under 600 lines).

## More examples in `examples/`

Around 40 standalone programs, each building to its own UF2. Some
particularly fun ones to try once you have the toolchain working:

- `examples/multicore_full_usb_demo.S` — both M33 cores running, one
  blinking the LED, the other writing to USB CDC, communicating via
  spinlock-protected shared state and an interpolator.
- `examples/pio_blink_demo.S` — a 9-instruction PIO program that
  blinks the LED *without* the CPU's help.
- `examples/sched_usb_demo.S` — the NVIC-priority scheduler driving a
  timer task that pushes bytes through an SPSC queue to a USB
  consumer.
- `examples/watchdog_usb_demo.S` — feed-the-watchdog, then stop, then
  watch the chip reset itself.

The Makefile knows about every `.S` in `examples/`; `make` builds
them all. `make build/NAME_flash.uf2` builds the flash variant of one.

## The test harness

rp-asm has a four-tier test setup:

- **T1** (`tests/unicorn/`) — Python tests that run each driver
  function inside the Unicorn emulator and verify the exact sequence
  of register writes it produces. Fast (milliseconds per test) and
  great for catching regressions.
- **T2** (`tests/qemu/`) — generic Cortex-M33 sanity inside QEMU.
- **T3** (`tests/renode/`) — full peripheral models in Renode for
  end-to-end interaction tests.
- **T4** — manual hardware verification on a real Pico 2.

If you ever want to add a feature: add a unicorn test first, watch
it fail, write the asm to make it pass. The harness is genuinely good
to develop against.

## C and Rust bridges

If you want assembly drivers but a higher-level language for the app
on top:

- **`c_bridge/`** + **`c_apps/hello_c/`** — write `main` in C, call
  `gpio_led_toggle` and friends through AAPCS. `docs/c_bridge.md`.
- **`rust_bridge/`** + **`rust_apps/hello_rust/`** — same for Rust,
  via a `no_std` crate that links to `librp_asm.a`.
  `docs/rust_bridge.md`.

Both demonstrate that AAPCS isn't just an academic convention; it's
the *real* contract that lets pure-asm drivers compose with any
ABI-compatible language.

## Inside this book

Two appendices that you'll probably keep open while you work:

- **[Appendix A — Glossary](A-glossary.md).** Every term we've used,
  defined once and indexed. When you forget what AAPCS or VTOR stands
  for, look here.
- **[Appendix B — Cheat sheet](B-cheat-sheet.md).** Every instruction,
  directive, and idiom we've used, in one printable page.

## Further reading

- **The RP2350 datasheet** — Raspberry Pi's main reference. About 1300
  pages, free PDF. Search for "RP2350 datasheet". The peripheral
  chapters are very readable; the system-level ones less so. Keep it
  open while you write drivers.
- **The ARMv8-M Architecture Reference Manual** — the formal
  specification of the instruction set. Dense but authoritative. Free
  from ARM with a (free) developer account.
- **The Cortex-M33 Technical Reference Manual** — implementation-level
  details of the specific core in the RP2350.
- **The ARM Cortex-M Generic User Guide** — the friendly,
  programmer-oriented summary of the architecture. This is the one to
  read first.
- **The picobin spec** — Raspberry Pi's format for the IMAGE_DEF
  block in your binary. See `docs/boot.md`.
- **The pico-sdk** (in C, on GitHub) — even though it's C, the
  pico-sdk's peripheral code is a useful cross-reference when the
  datasheet is ambiguous. Sometimes you'll spot an erratum workaround
  in the pico-sdk and want to mirror it in rp-asm.

## A few project ideas

If you'd like a target to learn against, in rough order of difficulty:

1. **Larson scanner.** Wire 8 LEDs to GP0..GP7 (each through ~330 Ω).
   Cycle a "bouncing" dot back and forth, driven by a timer.
2. **UART echo.** Receive bytes on UART0 RX, send them back on TX.
   Then add a uppercase-conversion mode triggered by a special byte.
3. **PWM-driven LED breathing.** Use the PWM driver to vary GP25's
   brightness in a sinusoidal pattern. The chip has no `sin`, so look
   up "tiny sine table" or use a piecewise-linear approximation.
4. **I²C temperature sensor.** Wire a BMP280, MCP9808, or any I²C
   sensor you have to GP2/GP3. Read the temperature; print it over
   UART. (The hardest part is reading the sensor's datasheet to find
   the right register sequence.)
5. **PIO logic analyser.** Use a PIO state machine to sample GP10 at
   ~10 MHz into a DMA-driven ring buffer, then dump the buffer over
   USB. This is a small but real engineering project that exercises
   PIO, DMA, and USB at once.

When you've done a few of those, you'll be a real embedded
programmer.

## Closing thought

There is something deeply clarifying about writing assembly on a
microcontroller. There is no language runtime hiding behaviour. There
is no OS scheduling your code. There is the silicon, and there is
the code, and the silicon does *exactly and only* what the code says.

That clarity makes you a better programmer in every language. The
next time you write a one-line Python expression that loads a CSV,
joins three tables, and writes a chart, you'll have a feeling — a
real, mechanical sense — for the millions of instructions that
expression is going to become. You'll know that, somewhere down at
the bottom of the stack, *somebody* had to do exactly the kind of
work you've been doing in this book.

You'll also have a Pico 2 that can blink an LED, talk to a serial
terminal, fade an LED with PWM, read a temperature sensor, and
generate a true random number — and you'll know every byte of how.

That's not bad for a 728-byte starting point.

Happy hacking.
