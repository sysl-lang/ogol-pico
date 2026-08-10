# ogol-pico

[Ogol](https://github.com/sysl-lang/ogol) on the **original** Raspberry Pi Pico W, over the USB serial
port — the same program as [the hosted console](https://github.com/sysl-lang/ogol-host) and as
[the Pico 2 W's](https://github.com/sysl-lang/ogol-pico2), differing only in where its bytes come from
and which chip it is built for.

```
Ogol on a Raspberry Pi Pico W. Type a line, or 'bye' to leave.

ogol> print 3 + 4
7
ogol> set a 5
ogol> print a
5
```

Echo, backspace, `←→`, Home, End, Delete, `Ctrl-A/E/B/F/U/K` and a history on the arrows — none of
which the port supplies, because a USB CDC connection has no line discipline. All of it is
`sysl.term.edit`, which is why this program is short.

## What is actually here

Almost nothing, and that is the point:

| | |
|---|---|
| the language | [`sh.sysl.ogol`](https://github.com/sysl-lang/ogol), a coordinate — including `session`, the read-run-print loop |
| the board | [`sh.sysl.pico`](https://github.com/sysl-lang/pico), a coordinate — `console_in()` and `console_out()` are the port as a `Reader` and a `Writer` |
| the editor | `sysl.term.edit`, in sysl's standard library |
| here | which streams to build, and a banner |

**There is no C in this project at all.** sysl exports `main`, the SDK's `crt0` branches straight into
it, and the archive in `add_executable`'s source list is what satisfies CMake's "a target must have
sources" rule with no translation unit to infer a language from.

## The difference from `ogol-pico2`, in full

Three lines of source and two of CMake — which is the argument for the shape rather than an
observation about it. `session` living in the language's own package is what keeps it to three.

| | `ogol-pico2` | here |
|---|---|---|
| import | `sh.sysl.pico2` | `sh.sysl.pico` |
| coordinate | `pico2` | `pico` |
| target | `thumb-freestanding-softfp` (Armv8-M, Cortex-M33) | `thumbv6m-freestanding` (Armv6-M, Cortex-M0+) |
| `PICO_BOARD` | `pico2_w` | `pico_w` |

**Getting the target wrong is not a link error.** An Armv8-M image on an M0+ faults at whatever
executes first, so check the ELF says so: `arm-none-eabi-readelf -A build/sysl_ogol.elf` should answer
`Tag_CPU_arch: v6S-M`, and `picotool info build/sysl_ogol.uf2` should answer `family ID 'rp2040'`.

**No `PICO_HARD_FLOAT_ABI`, unlike the Pico 2's earlier projects.** The M0+ has no FPU at all, so soft
float is not a convention chosen over hardware — it is the only thing there is, and sysl's
`thumbv6m-freestanding` and the SDK's default already agree.

## Building

Needs **sysl 0.0.40 or newer** — that is the release Armv6-M arrived in — the
[pico-sdk](https://github.com/raspberrypi/pico-sdk), `picotool`, and Arm's toolchain from the
**`gcc-arm-embedded` cask**; Homebrew's `arm-none-eabi-gcc` *formula* ships no newlib and cannot link
the SDK's own boot stage.

```
PICO_SDK_PATH=/path/to/pico-sdk cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
picotool load -f build/sysl_ogol.uf2
```

Then `screen /dev/cu.usbmodem<n> 115200`.

`-f` reflashes a running board with no button, because firmware built with `pico_enable_stdio_usb`
carries the USB reset interface. A board that has never carried such firmware needs BOOTSEL: unplug,
hold the button, plug in, release. **It mounts as `/Volumes/RPI-RP2`** — `RP2350` is the Pico 2's
name, and looking for that here finds nothing.

## Licence

ISC.
