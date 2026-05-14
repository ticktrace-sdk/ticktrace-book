# Appendix B — Cheat sheet

Printable one-pager. Everything you need to read or write a rp-asm
function.

## Register file (AAPCS roles)

| Reg | Alias | Role | Preserved? |
| --- | --- | --- | --- |
| r0 | — | arg 1 / return (low) | caller |
| r1 | — | arg 2 / return (high) | caller |
| r2 | — | arg 3 | caller |
| r3 | — | arg 4 | caller |
| r4–r11 | — | general-purpose | **callee** |
| r12 | ip | intra-procedure scratch | caller |
| r13 | sp | stack pointer | special, 8-byte aligned |
| r14 | lr | return address | clobbered by `bl` |
| r15 | pc | program counter | — |

Flags: **N** Z **C** **V** (negative, zero, carry, overflow).
Set by instructions ending in `s` (`adds`, `subs`, `lsls`, `cmp`, …).

## File header

```asm
    .include "rp2350.inc"           @ optional
    .syntax unified
    .cpu    cortex-m33
    .thumb
```

## Function template

```asm
    .section .text.NAME, "ax"
    .thumb_func
    .global  NAME
NAME:
    push    {r4, r5, lr}            @ if needed
    @ args in r0..r3, work, leave return in r0
    pop     {r4, r5, pc}
```

Leaf function (no `bl`, no callee-saved): drop the push/pop, end with
`bx lr`.

## Common instructions

### Move / load constants

```asm
    movs    r0, #25                 @ small immediate (<256)
    mov     r0, r1                  @ register-to-register
    ldr     r0, =0x40028000         @ any 32-bit constant
    ldr     r0, =symbol             @ address of a label
```

### Arithmetic

```asm
    adds    r0, r1, r2              @ r0 = r1 + r2 (flags set)
    subs    r0, #1                  @ r0 -= 1
    muls    r0, r1, r0
    sdiv / udiv  r0, r1, r2         @ signed / unsigned divide
```

### Bit operations

```asm
    ands / orrs / eors / bics  r0, r1
    lsls    r0, r1, #N              @ left shift
    lsrs    r0, r1, #N              @ logical right shift
    asrs    r0, r1, #N              @ arithmetic right shift
```

### Memory

```asm
    ldr     r0, [r1]                @ load word
    ldr     r0, [r1, #4]            @ word at r1+4
    ldr     r0, [r1, r2]            @ word at r1+r2
    str     r0, [r1, #N]            @ store word
    ldrb / ldrh / strb / strh       @ byte / halfword
    push    {r4, lr}                @ save
    pop     {r4, pc}                @ restore + return
```

### Compare / branch

```asm
    cmp     r0, r1
    beq / bne / blt / bgt / blo / bhs   .Llabel
    cbz     r0, .Llabel             @ if r0 == 0, branch
    cbnz    r0, .Llabel
    b       .Llabel                 @ unconditional
    bl      func                    @ call
    bx      lr                      @ return (leaf)
```

### Interrupts

```asm
    cpsid   i                       @ disable IRQs (critical section)
    cpsie   i                       @ re-enable
    wfi                             @ sleep until IRQ
```

## RP2350 atomic alias offsets

| Offset | Effect |
| --- | --- |
| `+0x0000` | normal read/write |
| `+0x1000` | atomic XOR (toggle set bits) |
| `+0x2000` | atomic SET (set set bits) |
| `+0x3000` | atomic CLR (clear set bits) |

## Common base addresses

| Name | Address |
| --- | --- |
| Bootrom | `0x00000000` |
| Flash (XIP) | `0x10000000` |
| SRAM | `0x20000000` |
| IO_BANK0 | `0x40028000` |
| PADS_BANK0 | `0x40038000` |
| UART0 | `0x40070000` |
| UART1 | `0x40078000` |
| I2C0 / I2C1 | `0x40090000` / `0x40098000` |
| SPI0 / SPI1 | `0x40080000` / `0x40088000` |
| TIMER0 / TIMER1 | `0x400B0000` / `0x400B8000` |
| RESETS | `0x40020000` |
| SIO | `0xD0000000` |
| NVIC | `0xE000E100` |

## GPIO idioms

```asm
    @ Drive GP25 high (set bit 25 in SIO_GPIO_OUT_SET)
    ldr     r0, =SIO_BASE + SIO_GPIO_OUT_SET_OFFS
    movs    r1, #1
    lsls    r1, #25
    str     r1, [r0]

    @ Toggle GP25 atomically
    ldr     r0, =SIO_BASE + SIO_GPIO_OUT_XOR_OFFS
    movs    r1, #1
    lsls    r1, #25
    str     r1, [r0]

    @ Read GP25 (returns 0 or 1 in r0)
    ldr     r0, =SIO_BASE + SIO_GPIO_IN_OFFS
    ldr     r0, [r0]
    lsrs    r0, #25
    movs    r1, #1
    ands    r0, r1
```

## UART idiom

```asm
    @ Send the byte in r1 over UART0, spinning if FIFO full
    ldr     r2, =UART0_BASE
1:  ldr     r3, [r2, #UART_FR_OFFS]
    tst     r3, #UART_FR_TXFF
    bne     1b
    str     r1, [r2, #UART_DR_OFFS]
```

## Interrupt idiom

```asm
    @ Install a handler and enable an IRQ line
    ldr     r1, =my_isr
    movs    r0, #IRQ_NUMBER
    bl      nvic_install_handler
    movs    r0, #IRQ_NUMBER
    bl      nvic_enable

    @ ISR shape
my_isr:
    push    {r4, lr}
    bl      peripheral_clear_irq
    @ ... handle event ...
    pop     {r4, pc}
```

## Scheduler idioms (`src/sched.S`)

```asm
    @ One-time setup
    bl      sched_init
    movs    r0, #TASK_ID          @ task id
    ldr     r1, =my_task          @ handler fn
    movs    r2, #PRIO             @ NVIC priority (lower = higher prio)
    bl      task_create

    @ Post a task — typically from an ISR (~5 cycles)
    movs    r0, #TASK_ID
    bl      task_post

    @ Critical section — disable all IRQs briefly
    bl      critical_enter        @ r0 = saved PRIMASK
    @ ... shared state mutation ...
    bl      critical_exit         @ r0 = saved value to restore

    @ sched_run never returns
    b       sched_run
```

SPSC byte queue (`src/spsc.S`):

```asm
    ldr     r0, =my_queue
    movs    r1, #'A'
    bl      spsc_byte_push        @ producer

    ldr     r0, =my_queue
    bl      spsc_byte_pop         @ consumer: r0 = byte, or -1 if empty
```

## Multicore idioms (`src/multicore.S`, `src/spinlock.S`)

```asm
    @ Launch core 1 (call once from core 0)
    ldr     r0, =core1_entry
    ldr     r1, =_core1_stack_top
    ldr     r2, =_ram_vectors
    bl      multicore_launch_core1

    @ FIFO mailbox between cores
    movs    r0, #0x42
    bl      multicore_fifo_push_blocking
    bl      multicore_fifo_pop_blocking   @ r0 = received word

    @ Hardware spinlock
    movs    r0, #0
    bl      spin_lock_blocking
    @ ... critical section ...
    movs    r0, #0
    bl      spin_unlock
```

Core 1 prologue (always required at the top of `core1_entry`):

```asm
core1_entry:
    @ Enable CP7 (RCP) on core 1's banked CPACR
    ldr     r0, =0xE000ED88
    movs    r1, #0xC0
    lsls    r1, r1, #8
    str     r1, [r0]
    @ Seed RCP canary (see examples/multicore_usb_demo.S:90-105)
    @ Zero MSPLIM
    movs    r0, #0
    msr     msplim, r0
    @ ... your code ...
```

## Build commands

```console
$ make                              # build everything, SRAM variants
$ make build/blinky_flash.uf2       # flash variant of one example
$ make test                         # T1 + T2 emulation tests
$ make test-all                     # adds T3 (Renode)
$ make pydeps                       # one-shot Python env setup
```

## Flashing

1. Hold **BOOTSEL** while plugging USB in.
2. Pico mounts as `RPI-RP2`.
3. Drag `build/NAME_flash.uf2` onto it.
4. Watch UART0 TX (GP0) at 115200 8N1.

## Local label conventions

| Form | Visible to linker? | Reusable? |
| --- | --- | --- |
| `main:` | yes (with `.global`) | no |
| `.Lloop:` | no | no |
| `1:` (numeric) | no | yes |

Numeric local labels are referenced as `1f` (forward) or `1b`
(backward).

## Reading raw output

```console
$ arm-none-eabi-objdump -d build/NAME.elf | less
$ arm-none-eabi-size build/NAME.elf
$ arm-none-eabi-nm build/NAME.elf | sort
```
