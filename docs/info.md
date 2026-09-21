<!---

This file is used to generate your project datasheet. Please fill in the information below and delete any unused
sections.

You can also include images in this folder and reference them in the markdown. Each image must be less than
512 kb in size, and the combined size of all images must be less than 1 MB.
-->

# How it works

This tile is **PRISM**, a Programmable Reconfigurable Indexed State Machine: a
protocol engine whose states, transitions, inputs and outputs are loaded at
run time as a bitstream compiled from an ordinary Verilog state machine.  A
small RISC-V core, TinyQV, sits beside it as the host processor.  PRISM does
the bit- and byte-level timing of a protocol in hardware, one state
transition per clock; the CPU loads the state machine, configures the
datapath, moves data through the FIFOs and runs the protocol's upper layers.
Protocols that have run on it in simulation include UART, SPI (slave, and
single- or quad-lane master), I2C (master and slave), 1-Wire, WS2812, a
quadrature encoder, a 24-bit GPIO expander, a USB low-speed device, and
10BASE-T Ethernet transmit and receive at once.

![](prism_periph.png)

## What is on the tile

- **The PRISM core**: 32 states, each described by a 128-bit State Execution
  Word (STEW).  Six input muxes pick from 32 inputs; two 3-input LUTs form an
  `if / else if / else` decision per state; each branch drives its own set of
  21 outputs; two more 2-input LUTs drive conditional outputs that follow the
  inputs within a state.
- **Two shards**: the machine can be fractured into two independent 16-state
  state machines, each with its own state index, inputs, outputs, datapath,
  FIFOs and host interrupt, talking to each other through semaphores.
  Unfractured, one 32-state machine owns everything.
- **A datapath per shard**: a 24/32-bit counter that doubles as a wide shift
  register, an 8-bit counter with compare, an 8-bit communication shifter
  (1 to 4 bits per shift), four constants and a constant table, a 64-byte
  FIFO, a 2 KB SRAM FIFO, a CRC unit (8/16/32) that doubles as a 32-bit
  up/down counter with compare, a Manchester bit recoverer, an edge-clocked
  sampler, edge-capture flops and a second timer.
- **CFGMEM**: eight latch-array macros (16 rows x 32 bits each) built with a
  DFFRAM-style flow hold the state table; the CPU loads them through a shift
  chain per bank.
- **An execution tracer** that records every clock's state, decision inputs
  and outcome into the SRAMs, and a **debugger** with halt, single step and
  condition-qualified breakpoints per shard.
- **TinyQV**: RV32EC + Zcb + Zicond, code from a QSPI flash and data in QSPI
  PSRAM, with a UART, a debug UART and GPIO.

The full register-level description of PRISM is in [prism.md](prism.md); the
design record behind it, decision by decision, is
[prism_interface.md](prism_interface.md).

## Pins

| Pin | Use |
| --- | --- |
| ui_in[6:0] | PRISM inputs 0-6 (each shard chooses raw, one-flop or two-flop synchronised) |
| ui_in[7] | TinyQV UART RX (or ui_in[3], by register) |
| uo_out[7:1] | PRISM outputs: each pin is routed by the running chroma to one of its four pin outputs, a conditional output or the shifter bit, or left to GPIO |
| uo_out[0] | TinyQV UART TX |
| uo_out[6] | doubles as the debug UART when selected |
| uio[7:0] | QSPI flash and PSRAM (bidirectional PMOD) |

## Memory map (host view)

| Address range | Device |
| ------------- | ------ |
| 0x0000000 - 0x0FFFFFF | Flash |
| 0x1000000 - 0x17FFFFF | RAM A |
| 0x1800000 - 0x1FFFFFF | RAM B |
| 0x8000000 - 0x800003F | TinyQV debug and time |
| 0x8000040 - 0x800007F | GPIO |
| 0x8000080 - 0x80000FF | UART |
| 0x8000100 - 0x800013F | PRISM CFGMEM programming (state table loader) |
| 0x8000200 - 0x80003FF | PRISM: common block, shard 0 window at +0x100, shard 1 window at +0x180 |
| 0xFFFFF00 - 0xFFFFF07 | TIME |

PRISM's two shard interrupts are TinyQV user interrupts 8 and 9.

## TinyQV, the host processor

TinyQV is a small RISC-V CPU designed for Tiny Tapeout.  It implements the
RV32EC instruction set plus the Zcb and Zicond extensions, with a couple of
caveats: addresses are 28 bits, program addresses 24 bits, and `gp` / `tp`
are hardcoded to 0x1000400 / 0x8000000.  Instructions are read over QSPI
from flash and a QSPI PSRAM holds data; the two share the QSPI clock and
data lines, so only one is accessed at a time.  Code runs from flash only.
In this design its job is to be PRISM's host: load a chroma, program the
datapath, feed and drain the FIFOs, service the interrupts.

### DEBUG

| Register | Address | Description |
| -------- | ------- | ----------- |
| SEL      | 0x800000C (R/W) | Bits 6-7 enable peripheral output on out6-7, otherwise out6-7 carry debug signals |
| DEBUG_UART_DATA | 0x8000018 (W) | Transmits the byte on the debug UART |
| STATUS   | 0x800001C (R) | Bit 0: debug UART TX busy |

See also [debug docs](debug.md).

### TIME

| Register | Address | Description |
| -------- | ------- | ----------- |
| MTIME    | 0xFFFFF00 (RW) | Get/set the 1 MHz time count |
| MTIMECMP | 0xFFFFF04 (RW) | Get/set the time to trigger the timer interrupt |

A 32-bit timer in the spirit of the RISC-V timer: MTIME advances at 1/64 of
the clock (nominally 1 MHz) and the interrupt asserts while MTIME is past
MTIMECMP by less than 2^30 microseconds.

### GPIO

| Register | Address | Description |
| -------- | ------- | ----------- |
| OUT | 0x8000040 (RW) | out0-7 when the GPIO function is selected |
| IN  | 0x8000044 (R) | The current state of in0-7 |
| FUNC_SEL | 0x8000060 - 0x800007F (RW) | Function select for out0-7 |

| Function Select | Peripheral |
| --------------- | ---------- |
| 0 | Disabled |
| 1 | GPIO |
| 2 | UART |
| 8 | PRISM (a pin PRISM does not claim falls back to GPIO) |

### UART

| Register | Address | Description |
| -------- | ------- | ----------- |
| TX_DATA | 0x8000080 (W) | Transmits the byte |
| RX_DATA | 0x8000080 (R) | Reads any received byte |
| TX_BUSY | 0x8000084 (R) | Bit 0: TX busy; bit 1: a received byte is available |
| DIVIDER | 0x8000088 (R/W) | 13-bit clock divider for the baud rate |
| RX_SELECT | 0x800008C (R/W) | UART RX pin: `ui_in[7]` when 0 (default), `ui_in[3]` when 1 |

# How to test

**Bring-up** is TinyQV's: load a program image into flash, reset (rst_n high
then low so the design sees a falling edge; program the flash and leave it in
continuous read mode and the PSRAMs in QPI mode; drive the QSPI chip selects
high with SD1:SD0 set to the read latency in cycles, 2 at 64 MHz; clock at
least 8 times and stop with the clock high; release the QSPI lines; raise
rst_n; then clock normally).  The RP2040 on the Tiny Tapeout demo board does
this from MicroPython.  Build programs with the
[customised toolchain](https://github.com/MichaelBell/riscv-gnu-toolchain)
and the [tinyQV-sdk](https://github.com/kdp1965/tinyQV-sdk), whose `prism.h`
knows this tile's register map (`PRISM_CONFIG` janestreet).

**A protocol** is then a chroma, a Verilog Mealy state machine compiled by
the [yosys-prism](https://github.com/kdp1965/yosys-prism) backend into the
32 x 128-bit state table plus the datapath configuration words.  The
sequence, in software or from the test bench:

1. Write the chroma from `chromas/`, or take one of the seventeen there:
   `make` in that directory compiles them all (`make yosys` fetches and
   builds the compiler first).
2. Load the table through the CFGMEM loader (`prism_load_chroma()` in the
   SDK: one word write per row with the bank's bypass bit set), program the
   shard's CFG0 / PINMUX from the compiler's output, set the datapath
   registers the chroma needs (preload, compare, constants, FIFO direction
   and levels), and set the enable bit.  The machine starts in state 0.
3. Talk to it through the shard window: host_in bits for handshakes, the
   FIFO for data, the interrupt for completion, the debugger to halt, step
   and break, the tracer to record what it did.

**The test suite** is the reference for all of this: `make -C test` runs the
cocotb tests (`test/user_peripherals/prism/`), twenty-six of them, each a
chroma driven by a model of the device on the other side of the wire: the
GPIO expander, SPI slave and master, UART, I2C master and slave, 1-Wire,
WS2812, the quadrature encoder, the PIO logic analyser and generator, a USB
low-speed host talking to the device chroma, an Ethernet decoder for the
transmitter, an encoder for the receiver, and a transmit-to-receive loopback
across the two shards; plus the fractured mode, the FIFOs (flop, SRAM and
32-bit access), the constant table, the second timer, the sampler, the
counter mode, the tracer against a golden record, and the debugger.  The same
suite runs on the gate-level netlist.

# External hardware

The design expects the [QSPI PMOD](https://github.com/mole99/qspi-pmod) on
the bidirectional PMOD (16 MB flash, two 8 MB RAMs); the UART is on the pins
of the demo board's RP2040 hardware UART.

Beyond that, whatever the protocol needs: PRISM drives logic levels on
uo_out and reads them on ui_in.  A USB low-speed device wants the usual
series resistors and a 1.5 kOhm pull-up on D-, with D+ / D- on two outputs
and an output-enable pin driving an external tri-state buffer.  10BASE-T
wants magnetics with a differential driver on the transmit pair and a
receiver (a comparator is enough) on the receive pair.  I2C wants external
open-drain buffers, since the tile's outputs are push-pull.  Buttons on the
inputs are handy for the host-in handshakes while developing.
