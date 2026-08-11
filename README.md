# rp2040blocks

The RP2040's peripheral blocks for sysl — the smallest useful thing on top of the register map. GPIO,
SPI, and the memory-mapped-register primitives both rest on.

```hocon
dependencies {
  rp2040blocks { git = "github.com/sysl-lang/rp2040blocks", version = "0.1.0" }
}
```

```sysl
import sh.sysl.rp2040blocks.gpio
import sh.sysl.rp2040blocks.spi

// A display: clock and data to the SPI block, chip-select and data/command to the processor.
gpio.select(2, gpio.SPI)
gpio.select(3, gpio.SPI)
gpio.output(5)
gpio.output(6)

val bus = spi.spi0()

spi.up(bus, 125_000_000, 31_250_000)

gpio.low(5)
gpio.low(6)
spi.send(bus, command)
spi.drain(bus)          // BSY, not the FIFO — see below
gpio.high(6)
spi.send(bus, pixels)
spi.drain(bus)
gpio.high(5)
```

## Why this is not part of `rp2040`

**`sh.sysl.rp2040` is generated and must not be edited.** Every one of its `.sysl` files comes out of
Raspberry Pi's own `RP2040.svd` by way of [sysl-lang/svd](https://github.com/sysl-lang/svd), and its
README says a hand edit is lost the next time the chip is regenerated. What that buys is a package
you can regenerate and diff to nothing; what it costs is that there is nowhere in it for a line of
code to live. This is that nowhere.

The split is the ordinary one — a generated register description underneath, hand-written bring-up on
top — and the names say which is which. `rp2040` describes registers; `rp2040blocks` drives the blocks
they belong to. **"Block" is the datasheet's word and the SDK's**, which is the point of choosing it:
`reset_block`, `unreset_block_wait`.

## What it is not

**Not a HAL.** There is no portable abstraction here and no attempt at one — every function names an
RP2040 peripheral and does what that peripheral's chapter says. A second chip would get its own
package rather than an implementation of an interface.

**Not the pico-sdk.** Nothing here calls a C function, so a program using this package needs no SDK,
no CMake and no C toolchain — `sysl build` alone will do it. `sh.sysl.pico` is the other choice and a
different kind of thing: it is the SDK's surface, it brings the wireless chip and USB stdio with it,
and it cannot compile without the SDK's headers. The two compose — `picokit` uses both.

**Not complete.** GPIO and SPI are what one real program needed. I2C, ADC, PWM, DMA and PIO are all
absent, and each is a small piece of work against a chapter of the datasheet.

## The three things worth reading before using it

**A peripheral comes out of reset before it is configured.** Every register reads and writes
perfectly while its block is held in reset, and none of it does anything — so a driver that skips
this configures nothing and reports no error. `spi.up` does it; anything added here must too.

**`drain` waits on `BSY`, not on the FIFO being empty.** An empty transmit FIFO says the block has
*accepted* the bytes, not that it has *sent* them. A display driver that lowers its data/command pin
on an empty FIFO turns its last parameter into a command, which the panel obeys — and it works at low
clock rates, which is the worst way for a bug like this to behave.

**`spi.send` reads back every byte it writes.** Every frame clocked out clocks one in, and a receive
FIFO nobody empties overflows, after which the block keeps running while its status bits stop meaning
what the loop reads them to mean.

## The atomic aliases

Every peripheral register on this chip has three more addresses above it, and writing a mask to one
of them is a read-modify-write the hardware does indivisibly:

| alias | offset | writing a mask |
|---|---|---|
| `mmio.xor` | `0x1000` | flips those bits |
| `mmio.set` | `0x2000` | raises those bits |
| `mmio.clr` | `0x3000` | lowers those bits |

**This is not a convenience.** A read-modify-write done in software races the other core, and the
registers most worth changing one bit of — `RESETS`, the pad controls — are exactly the ones both
cores reach.

`SIO` is the one block with no aliases, because it is per-core and single-cycle by construction; its
own `SET`/`CLR`/`XOR` registers do the same job and `gpio` uses them.

## Function numbers are the same on every pin

`sh.sysl.rp2040` generates a constant per pin per function — `GPIO2_CTRL_FUNCSEL_SPI0_SCLK`,
`GPIO3_CTRL_FUNCSEL_SPI0_TX` — because that is what the SVD describes. All of them for a given
function are the same number, so `gpio` names the function once: `gpio.SPI`, `gpio.UART`, `gpio.I2C`,
`gpio.PWM`, `gpio.SIO`, `gpio.PIO0`, `gpio.PIO1`, `gpio.CLOCK`, `gpio.USB`.

**`gpio.NONE` is 31 and not 0.** Zero is a real function on the pins that have one — JTAG, on GPIO 0
to 3 — so a pin "switched off" by writing zero is a pin quietly handed to the debug port.

## Tests

```
sysl test .
```

Seven of them, and they all test one function: the SPI bit-rate divisor search. **That is the honest
extent of what a host can check** — everything else here ends at an address on a chip that is not
present.

It is worth testing precisely because it is the one thing here that can be subtly wrong rather than
obviously wrong: a wrong base address fails immediately and visibly, while a bus running at the wrong
clock works on a short ribbon and fails on somebody else's board. The overflow guard has a test of
its own, because `(prescale + 2) * 256 * target` leaves 32 bits for any target above about 60 kHz and
the failure is a bus many times faster than asked for.

The rest of this package is proven the only way it can be — on hardware, by
[picokit](https://github.com/sysl-lang/picokit).

## Capabilities

None, and every module declares `@no_alloc`. A capability clause is a claim about what a library
*does*, and what this one does is arithmetic on addresses and stores through raw pointers. It opens
no file, reads no clock, asks an operating system for nothing, and never reaches a heap — which it
had better not, on a chip with 264 KB and no operating system to have one.

## Licence

ISC.
