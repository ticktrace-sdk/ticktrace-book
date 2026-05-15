# Appendix E: Studio

Up to this point every example in the book has been built and flashed
the same way: `make build/<name>_flash.uf2`, hold BOOTSEL, drag-drop.
That is the truthful, low-level path: the Makefile is the build
system, `picotool` (or the BOOTSEL drive) is the flasher, and you
read the resulting UF2 with your eyes if you ever need to debug what
went onto the chip. It is also the path you reach for when you want
to know exactly what the toolchain is doing.

Studio is the same engine wrapped in something a new user can drive
without reading a Makefile. You pick a target, toggle peripherals,
click Build, click Flash. Under the hood it runs the same
`arm-none-eabi-as` and `arm-none-eabi-ld` invocations, against the
same sources in `src/`, with the same linker scripts in `link/`.
There is no parallel build — no second copy of `uart.S` lives inside
Studio. The catalog points at `../src/uart.S` and that is the file
that gets assembled.

This appendix walks through using Studio for the workflows the
preceding chapters established by hand: building a flash blinky,
building an SRAM blinky, picking a custom set of peripherals,
shipping a bootloader chain, and pushing per-slot updates.

## When to use Studio (and when not to)

Studio is the right tool when:

- You're new to the SDK and want to see things build before you
  understand why.
- You're prototyping and want to flip peripherals on and off without
  editing example sources.
- You're shipping a bootloader chain (chapter material is in
  [`docs/bootloader.md`](../docs/bootloader.md)). The TOML field
  `[bootloader] tsbl = "ab"` is dramatically less typing than the
  full `make build/firmware_<name>.uf2` recipe.
- You're doing A/B field updates on a board that already has the
  bootloader installed.

The Makefile is still the right tool when:

- You want to know exactly which `as`/`ld` flags were used.
- You're writing CI: `make test-t1`, `make test-tools`, `make test-t2`
  is the canonical entry point.
- You need to script the build of *something Studio doesn't model
  yet* — a custom linker script, a one-off footer, a hand-rolled UF2.
- You are debugging the build itself.

The CLI half of Studio (`rpasm`, the binary, not the GUI) lives in
between. It's scriptable and exposes every operation the GUI does,
so you can pin it into CI alongside `make` once you trust it.

## Building blinky

The fastest way through the GUI is to use the example dropdown.
From the SDK root:

```
$ go run ./studio/cmd/rpasm-studio
```

A window opens with two mode tabs at the top: **Examples** and
**Custom Project**. Examples mode is selected by default, with
`blinky` (the canonical `src/main.S`) preselected. The Target
dropdown shows `rp2350-arm`; Layout defaults to `sram` (faster
iteration cycle, doesn't wear flash). Click **Build**. The Output
panel scrolls through the assemble/link steps and ends with three
paths: `ELF`, `BIN`, `UF2`. The Flash button (which was greyed out
as "Flash (build first)") activates.

Put the Pico 2 in BOOTSEL (hold the button, plug in the cable),
click **Flash**, and the on-board LED starts blinking. Same result
as `make build/blinky.uf2` followed by drag-drop — about 20 fewer
keystrokes, and the build log is sitting right there to read.

For a flash-resident build, flip the Layout dropdown to `flash`,
rebuild, reflash. Whatever was at `0x10000000` is overwritten with
the new image; the SRAM build no longer survives a power cycle but
the flash one does.

## Custom Project mode

Examples mode picks one source file from `examples/` and links it
against every default-on module from the catalog. That's enough to
get started but doesn't let you say "I want UART and PIO but not
DMA". Custom Project mode does.

Switch to the **Custom Project** tab. The left pane changes: where
Examples mode showed a preview of the picked `.S` file, Custom mode
shows a Source path input and a grid of Features checkboxes
organised by category. Each checkbox is one module in the catalog
(`STARTUP`, `UART`, `GPIO`, …); they start at their catalog default
(most are default-on).

Type or paste a source path: `../src/main.S` for the standard blinky,
or any of your own `.S` files. Uncheck the modules you don't need —
typically `USB`, `PIO`, `SHA256`, `TRNG`, etc., when the app doesn't
touch those peripherals. Just below the Source path input, a row of
"Selected modules" badges shows the live set that the next Build
will pull in. Click Build, then Flash, same as before.

The catalog itself is regular files at `studio/catalog/`. To add a
module you author a small TOML descriptor like
`studio/catalog/peripherals/myperiph.toml`:

```toml
symbol      = "MYPERIPH"
name        = "My Peripheral"
category    = "peripherals"
order       = 145
default     = false
description = "What this module does."
sources     = ["../src/myperiph.S"]
```

It shows up in the GUI's Features grid on the next launch. There's
no plugin system, no rebuild of Studio — the catalog is just data.

## Saving a project

The settings the GUI captures (target, layout, the source path, the
feature toggles) round-trip through a `.rpasm.toml` file via the
**Save** and **Load** buttons in the project row. Save once, and the
same configuration is one click away next time, or shareable as a
text file. The CLI consumes the same format
(`rpasm build path/to/project.rpasm.toml`), so a project file
authored in the GUI is what your CI runs against in the Makefile.

Loading a project also populates the path resolution: the GUI walks
up to find `go.mod` + `catalog/`, and **Browse...** opens a native
file dialog (zenity on Linux, the OS picker elsewhere) when you'd
rather click than type.

## The bootloader workflow

If you've read [`docs/bootloader.md`](../docs/bootloader.md) you
know the shape of the chain: bootrom → SSBL → TSBL → app slot. From
the Makefile, building that whole stack into one UF2 takes a
`make bootloader && make build/blinky_app.elf && make build/firmware_blinky.uf2`
incantation. From Studio it's one TOML field.

Add to your project TOML:

```toml
[bootloader]
tsbl = "ab"            # "bypass" for single-slot, "ab" for A/B + rollback
```

Studio sees the field and switches the build engine into chain mode:

1. App links at slot A's base (`0x10008000`) using
   `link/app_at_0x10008000.ld`.
2. SSBL is assembled from `src/ssbl/ssbl.S` + `src/crc32.S`, linked
   with `link/ssbl.ld`.
3. The chosen TSBL flavor (`src/tsbl/tsbl_ab.S` or `tsbl_bypass.S`)
   is assembled and linked with `link/tsbl.ld`.
4. CRC32 + SHA-256 footers are computed for TSBL and app.
5. The pieces are stitched into `firmware_<name>.uf2` at their
   canonical addresses.

The Memory tab now has a third section, "Bootloader chain", that
breaks the firmware down by stage — SSBL / TSBL-`<flavor>` / Slot A /
Slot B — with each one's used and capacity in bytes. Slot B reads
as 0 B used after a full firmware flash: the chain UF2 only
populates slot A, and TSBL-ab will see only A as valid and boot it.

Flash the firmware UF2 with Flash as normal. The board boots through
the whole chain.

If the app needs to talk back to the TSBL — to confirm a successful
boot for the A/B rollback path, or to request a reboot into DFU mode
— enable the **Boot API** module in the Features grid. It pulls in
`src/boot_api.S`, which exposes `boot_confirm()`,
`boot_request_dfu()`, and `boot_request_bootsel()` to your app code.

## Per-slot field updates

Once the bootloader chain is in place, you rarely re-flash the whole
thing. The usual update pattern is "the device is running, push a
new app to the *other* slot, let TSBL-ab pick it up on the next
boot, and roll back if it's buggy."

Studio's per-slot flag does exactly that:

```
$ rpasm flash --slot b path/to/project.rpasm.toml
```

This rebuilds the app at slot B's base (`0x10080000`) using the
slot-B linker script, packs a slot-only UF2 with just the app and
its footer, and writes only `0x10080000`/`0x100F7F00`. SSBL and TSBL
are untouched. The footer is written with `seq=2` so TSBL-ab's
higher-sequence-wins selector picks slot B over slot A on the next
boot.

If the new app calls `boot_confirm()`, the rollback marker in
`WATCHDOG_SCRATCH[6]` is cleared and slot B sticks. If it doesn't
— if it crashes, hangs, or triggers a watchdog reset before
confirming — the marker survives the warm reset, TSBL-ab sees it on
the way back up, and rolls back to slot A. There's a worked example
of the watchdog-driven rollback in `examples/blinky_buggy_demo.S`
and the rollback demo firmware (`make build/firmware_rollback_demo.uf2`).

The `--slot a` flag does the symmetric thing for slot A. Either
way, the bootloader chain is the part that *isn't* in the UF2 you
flash, which is what makes the update cheap and recoverable.

## Inspecting a board

`rpasm bootinfo` (or the GUI's **Query slots** button in the tools
row) puts the connected BOOTSEL board through an EXIT_XIP + READ
sequence, parses both slot footers, and reports them:

```
$ rpasm bootinfo
slot A @ 0x10008000: status=good seq=1 payload=1188 B crc=0xcbf6f7d1
slot B @ 0x10080000: status=good seq=2 payload=1188 B crc=0xa5e10efa
```

This is the field-debugging tool. If a customer reports their board
boots the old slot after an update, `bootinfo` is how you confirm
whether the update wrote where it claimed.

## The CLI under the GUI

The GUI is a Gio window over the same engine `rpasm` exposes from
the command line. Anything in the GUI maps to one or more CLI
calls:

| GUI action                         | CLI equivalent                                          |
| ---------------------------------- | ------------------------------------------------------- |
| Build button                       | `rpasm build project.toml`                              |
| Flash button                       | `rpasm flash project.toml`                              |
| Flash with `[bootloader]` set     | `rpasm flash project.toml` (auto picks firmware UF2)    |
| Reset to BOOTSEL                   | `rpasm reboot --bootsel`                                |
| Query slots                        | `rpasm bootinfo`                                        |
| Memory tab content                 | `rpasm build` tail (printed after the build summary)    |
| Validate without building          | `rpasm validate project.toml`                           |
| Toolchain + udev + board status   | `rpasm doctor`                                          |

If you have an existing Makefile-driven flow and want to add Studio
to it incrementally, the CLI is the surface to bind against. The
GUI is purely the user-facing front; everything is scriptable.

## When something doesn't work

The first thing to check is `rpasm doctor`:

```
$ rpasm doctor
root:    /home/.../rp-asm/studio
modules: 26
targets: 1

[target rp2350-arm]
  as:      /usr/bin/arm-none-eabi-as
           ...
[board]
  udev:    /etc/udev/rules.d/99-rpasmboot.rules present
  boards:  none in BOOTSEL right now ...
```

Three classes of problem this surfaces:

1. **Missing toolchain.** `as`/`ld`/`objcopy` paths fail to resolve
   to a real binary. Install `gcc-arm-none-eabi` (or equivalent on
   your distro), make sure the prefix in `catalog/targets/rp2350-arm.toml`
   matches what you installed.

2. **Missing udev rule.** On Linux, PICOBOOT USB access without
   the rule fails with `EACCES`; `rpasm flash --method rpasmboot`
   would fall back to drive-copy and you'd wonder why everything
   is slower. The doctor prints the one-line `sudo ... && sudo
   udevadm ...` command.

3. **No board.** "Hold BOOTSEL while plugging in the board" is
   nearly always the answer. If `lsusb` shows a `2e8a:000f` and
   doctor still says no boards, something is wrong with the udev
   rule installation.

For protocol-level problems with rpasmboot itself, the diagnostic
log lines mirror to the launching terminal — every line the GUI
shows in its Output panel also prints on stderr. So
`go run ./studio/cmd/rpasm-studio 2>&1 | tee studio.log` captures
everything for paste-into-issue debugging without needing a
selectable widget.

## Where to go from here

This appendix is the narrative tour. For the field-by-field
reference — every project TOML key, every CLI flag, every internal
package — see [`docs/studio.md`](../docs/studio.md). The bootloader
chain itself (memory map, footer layout, status state machine,
watchdog ABI) is in [`docs/bootloader.md`](../docs/bootloader.md).
The peripheral modules referenced by the catalog have their own
datasheets under `docs/{uart,usb,pio,...}.md`.

Studio is meant to dissolve into the background once you know what
it's doing. The Makefile, the linker scripts, the SDK sources — none
of those go away. Studio is just a cleaner front-end onto them.
