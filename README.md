# PRISM + TinyQV on IHP SG13G2, 8x4 tile with the SRAM FIFOs (ttihp26b trial)

This directory is the **full design on an ihp-sg13g2 8x4 tile**: the same PRISM +
TinyQV design as [ihp-um-janestreet-prism](../ihp-um-janestreet-prism) (both IHP
512x32 SRAM FIFOs, tracing), ported the way the 5x4 SRAM-less variant
[ihp-sg13g2-janestreet-prism](../ihp-sg13g2-janestreet-prism) was (it is a copy
of that directory; keep the three in sync by hand).  Differences from the 5x4
variant: `tiles: "8x4"`, `PRISM_SRAM_FIFO=2`, the SRAM macros from the PDK's
`sg13g2_sram` library at the CMOS5L positions, the CFGMEM columns at x 547.06
and 1359.68 (on the 38.87 um TopMetal1 stripe grid with the same macro builds),
`Project.DiodesOnSramPins`, density 52, and a local copy of the SRAM GDS in
`macros/` whose PR boundary is moved from layer 235/4 (older IHP release in the
local PDK, unknown to Magic and to the Tiny Tapeout layer whitelist) to 189/4.  The stripe step
(`odb_stripes.py`) moves the TopMetal1 tile stripes onto the SRAM's Metal4 power
columns and ties them down with Via4 + TopVia1 stacks (`EXTEND_STRIPES_SRAM_LAYER`).

What differs from the CMOS5L repository:

- **RTL**: every hardcoded IHP cell is a `` `IHP_<CELL> `` define
  (`IHP_AND2_1`, `IHP_OR2_2`, `IHP_DLHQ_1`, ...) that resolves to `sg13g2_*`
  when `IHP_SG13G2` is defined (`VERILOG_DEFINES` in `src/config.json`) and
  to `sg13cmos5l_*` otherwise; `src/tinyQV/cpu/latch_reg.v` takes its IHP
  latch/clock-gate branch for either `SCL_` define (`SCL_IHP`).
- **Config** (`src/config.json`): `tiles: "5x4"` in `info.yaml`
  (1071.84 x 710.64 um), no SRAM macro entries, `PRISM_SRAM_FIFO=0`, the
  Tiny Tapeout sg13g2 tile PDN (single layer: TopMetal1 vertical stripes at
  the template's 38.87 um pitch, `FP_PDN_MULTILAYER` 0; TopMetal2 is a
  forbidden layer for projects, so the PDK's default multi-layer grid fails
  the precheck), `Project.MouthKeepouts` and `Project.ExtendPowerStripes`
  (`EXTEND_STRIPES_LAYER` TopMetal1) substituted, `pdn_cfg.tcl` without the
  per-macro grid, density 70, the two CFGMEM columns flush against the core
  edges at x 2.88 and 737.76, `KLAYOUT_DRC_RUNSET` = the sg13g2 deck
  (`tools/drc_deck.py` swaps in the IHP dev-branch deck locally).
- **Macros** (`macros/`): `CFGMEM_IHP16` / `CFGMEM_IHP_LEFT16` built by
  DFFRAM for sg13g2 with Metal2 + Metal3 route-through and the Di0/Do0 pins
  on Metal4:
  `python3 dffram.py -p ihp-sg13g2 -s sg13g2_stdcell -b cfgmem_ihp[_left] [--left] --pins-to-metal4 '^D[io]0\[' --build-dir build/ihp-sg13g2 --products-dir products/ihp-sg13g2 16x32`.
  Their power pins are TopMetal1 rails on the tile's stripe grid (pitch
  38.87; `make cfgmem16_sg13g2_m4 left_sg13g2_m4 M4_PINS='^D[io]0\['` in
  DFFRAM passes `-c PDN_VOFFSET=15.81` / `12.16` so the rails coincide with
  the tile stripes at x 737.76 / 2.88), and the stripe-extension step runs
  the tile stripes across them exactly as on the CMOS5L tile.
- **Flow**: `flow.py`, the Makefile and the precheck target use
  `ihp-sg13g2`; `tt/` is tt-support-tools `main` (it ships the 5x4
  template, so there is no `tt-8x4` step).  `make config` then
  `python -m librelane --pdk ihp-sg13g2 --manual-pdk ... src/config_merged.json`
  in the LibreLane nix shell, or `make harden`.

Status (2026-09-18): `runs/g2_5x4b` hardens on the Mac in about 22 minutes end
to end (22.8k cells, 74% utilization, global-routing overflow 1.3k against 8k-12k
on the CMOS5L 8x4 tile, detailed routing clean after 7 passes), with antenna,
KLayout DRC and LVS clean and the Tiny Tapeout precheck passing
(`make precheck RUN_DIR=runs/g2_5x4b`).  Setup slack is +0.96 ns at the
typical corner and negative at the slow corner, as on the CMOS5L tile.

The rest of this file is the CMOS5L README.

---

# PRISM + TinyQV on IHP CMOS5L (Jane Street protocol-emulator ASIC entry)

This repository is the IHP SG13CMOS5L port of
[ttsky26a-qv-prism](../ttsky26a/ttsky26a-qv-prism): a PRISM programmable
state-machine engine driven by a TinyQV RISC-V SoC, with the PRISM
configuration held in `CFGMEM_IHP16` latch-array macros built by
[DFFRAM.librelane](../DFFRAM.librelane) (`make cfgmem16_cmos5l` /
`make left_cmos5l`; views in `macros/`). It starts from
`TinyTapeout/ttihp-verilog-template@cmos5l` (workflows, devcontainer, config
defaults) as the competition rules require.

Platform-specific RTL is guarded: `SIM` selects behavioural models,
`SCL_sky130_fd_sc_hd` the original sky130 cells, and the fall-through
branches use `sg13cmos5l_*` cells (same pin names). LibreLane defines the
`SCL_<library>` macro at synthesis.

## Hardening locally

The single-layer Tiny Tapeout PDN cannot reach the macros' Metal4 power pins
on its own (pdngen carves its stripes around macros), so the flow gets one
extra step, `Project.ExtendPowerStripes` (`odb_stripes.py`), that draws the
tile stripes across each macro's pin columns.  The step is a LibreLane
plugin (`librelane_plugin_prism_pdn.py`): LibreLane imports any
`librelane_plugin_*` module it finds on the Python path, and running
`python -m librelane` from the repository root, which is what
`tt_tool.py --harden` and therefore the Tiny Tapeout GDS action do, puts the
root on that path.  `meta.substituting_steps` in `src/config.json` then
inserts the step after `OpenROAD.GeneratePDN`.  Nothing has to be installed
and the CI build gets the same connected macro power as the local one.

The tile's stripes run on a 44.96 um pitch with 5.62 um between VPWR and
VGND: the IHP SRAM's power-column grid (4 x 11.24 um), which the CFGMEM
macros were regenerated to match, so one set of full-height stripes powers
the macros and the SRAM.  With `FP_PDN_VOFFSET` 6.15 the first VPWR stripe
is at 9.03 um; CFGMEM macros go at x = -2.41 + 44.96k um, the SRAM at
3.36 + 11.24i um, all on row boundaries (n x 3.78 um).

    git clone -b cmos-8x4 https://github.com/kdp1965/tt-support-tools tt   # once
    make venv          # once: Python environment for tt_tool.py (.venv-tt)
    make harden-tt     # exactly what CI runs: tt_tool.py --harden --ihp --no-docker
    make harden        # alternative: flow.py (same step, inserted in Python)

The tile is 8x4 (1724.16 x 710.64 um, the size allowed by the Jane Street
competition).  The upstream `cmos` branch of the TT tools has no 8x4 template;
the `cmos-8x4` branch of the fork adds ours from `tt_8x4/` (see its README),
and the GitHub workflows point the actions at that branch.  `make config`
also installs the template into an upstream checkout.  The CFGMEM macros form
two full-height columns of four: bank A (lo macros, `CFGMEM_IHP_LEFT16`,
pins facing east) at x = 537.11 um and bank B (hi macros, `CFGMEM_IHP16`,
pins facing west) at 1346.39 um, with the PRISM logic in the 478 um
channel between them.  The two 512x32 SRAM FIFO macros (one per shard,
2 KB each) are stacked flush left at the bottom with their pin faces
toward each other: (3.36, 3.78 um) `FS` and (3.36, 340.20 um) `N`, the
145 um gap between them holding both FIFO wrappers and the trace ports;
TinyQV lives above
the stack under the tile pins.  All the 1P x32 macros share one width
and the same power-column positions, so the stripes on the columns of
the lower one serve the upper one too.  A CFGMEM column is a routing wall (Metal2 blocked, Metal3 chopped),
so nothing but the SRAM and its wrapper may sit on the far side of one;
docs/prism_interface.md section 4d.1 records the experiments behind this
and the band-layout alternative kept in `src/config_band_v6.json`.

Results land in `runs/wokwi/final/`.  To look at any step's database in the
OpenROAD GUI, or the GDS in KLayout (both come from the nix shell):

    make gui FILE=runs/wokwi/22-project-extendpowerstripes/tt_um_pettit_js_prism.odb
    make gui RUN_DIR=runs/wokwi_phase6      # final database of another run
    make klayout                            # final GDS of runs/wokwi

---

![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg) ![](../../workflows/test/badge.svg) ![](../../workflows/fpga/badge.svg)

# TinyQV - A Risc-V SoC for Tiny Tapeout

![Risc-V competition logo](docs/riscv_compo.jpg)

TinyQV is accepting peripherals for tape out on the Tiny Tapeout [ttsky25a shuttle](https://app.tinytapeout.com/shuttles/ttsky25a) as part of the [Tiny Tapeout Risc-V peripheral challenge](https://tinytapeout.com/competitions/risc-v-peripheral/).

To contribute, start from either:
- The [byte peripheral template](https://github.com/TinyTapeout/tinyqv-byte-peripheral-template) for simpler peripherals, or
- The [full peripheral template](https://github.com/TinyTapeout/tinyqv-full-peripheral-template).

Further reading:
- [Documentation for project](docs/info.md)
- [More details about tinyQV](https://github.com/MichaelBell/tinyQV/tree/ttsky25a)
- [tinyQV-sdk for building tinyQV programs](https://github.com/MichaelBell/tinyQV-sdk)
- [Example tinyQV programs](https://github.com/MichaelBell/tinyQV-projects)
- [tinyQV Micropython](https://github.com/MichaelBell/micropython/tree/tinyqv-sky25a)

## TinyQV SoC Diagram

![TinyQV Soc Arhitecture Diagram](docs/architecture.png)

## Peripheral data transaction timing

### Read transactions

![Read transaction diagram](docs/wavedrom/read_txn.png)

`data_read_n` signals when there is a read and indicates the transaction width, encoded as in RV32 load instructions: 0, 1 or 2 for 8, 16 or 32-bit.  3 means no transaction.

The read may complete synchronously on the same clock, or be delayed by any number of clocks while the peripheral prepares the data.  `data_ready` signals when the transaction is complete.  `data_out` is sampled on the next clock, its value does not have to be held constant for any additional clocks.  Data for 8 or 16-bit reads should always be aligned to the LSB of `data_out`, even for unaligned reads.

The top diagram shows a synchronous transaction, the bottom diagram shows a delayed transaction.

Reads from the peripheral (loads to TinyQV) happen at most once every 24 clocks.  As long as data_ready is signalled within 7 clocks there is no impact on maximum instruction throughput.

### Write transactions

![Write transaction diagram](docs/wavedrom/write_txn.png)

Writes to the peripheral (stores from TinyQV) happen at most once every 8 clock cycles - the top diagram shows two writes as close together as possible.  The `address` is guaranteed to be stable for 8 clocks starting at the transaction.  Peripherals must accept writes, they can't delay the next transaction.

`data_write_n` signals when there is a write and indicates the transaction width, encoded as in RV32 store instructions: 0, 1 or 2 for 8, 16 or 32-bit.  3 means no transaction.

Data for 8 or 16-bit writes is aligned to the LSB of `data_in`, even for unaligned writes.

The `data_in` is modified between transactions, but due to the quad serial nature of TinyQV it is only modified 4 bits at a time, starting at the least significant bits.  Advanced users could rely on the upper bits being stable for additional clocks.

## What is Tiny Tapeout?

Tiny Tapeout is an educational project that aims to make it easier and cheaper than ever to get your digital and analog designs manufactured on a real chip.

To learn more and get started, visit https://tinytapeout.com.

## Set up your Verilog project

1. Add your Verilog files to the `src` folder.
2. Edit the [info.yaml](info.yaml) and update information about your project, paying special attention to the `source_files` and `top_module` properties. If you are upgrading an existing Tiny Tapeout project, check out our [online info.yaml migration tool](https://tinytapeout.github.io/tt-yaml-upgrade-tool/).
3. Edit [docs/info.md](docs/info.md) and add a description of your project.
4. Adapt the testbench to your design. See [test/README.md](test/README.md) for more information.

The GitHub action will automatically build the ASIC files using [OpenLane](https://www.zerotoasiccourse.com/terminology/openlane/).

## Enable GitHub actions to build the results page

- [Enabling GitHub Pages](https://tinytapeout.com/faq/#my-github-action-is-failing-on-the-pages-part)

## Resources

- [FAQ](https://tinytapeout.com/faq/)
- [Digital design lessons](https://tinytapeout.com/digital_design/)
- [Learn how semiconductors work](https://tinytapeout.com/siliwiz/)
- [Join the community](https://tinytapeout.com/discord)
- [Build your design locally](https://www.tinytapeout.com/guides/local-hardening/)

## What next?

- [Submit your design to the next shuttle](https://app.tinytapeout.com/).
- Edit [this README](README.md) and explain your design, how it works, and how to test it.
- Share your project on your social network of choice:
  - LinkedIn [#tinytapeout](https://www.linkedin.com/search/results/content/?keywords=%23tinytapeout) [@TinyTapeout](https://www.linkedin.com/company/100708654/)
  - Mastodon [#tinytapeout](https://chaos.social/tags/tinytapeout) [@matthewvenn](https://chaos.social/@matthewvenn)
  - X (formerly Twitter) [#tinytapeout](https://twitter.com/hashtag/tinytapeout) [@tinytapeout](https://twitter.com/tinytapeout)
