# xcbrup

Update the block RAM contents of a routed Xilinx 7-series design without running
place and route again.

This is the Xilinx counterpart of [`brup`](https://github.com/berke/brup): once
your design is placed and routed, a firmware change should cost you a synthesis
run and a couple of seconds, not another placement.

It is built for the [openXC7](https://github.com/openXC7) toolchain, because that
flow keeps the routed result in FASM, a text format where every block RAM bit has
a name. That single property does most of the work. `brup` has to locate memory
contents inside an anonymous ECP5 configuration blob; here the bits are already
labelled:

```text
BRAM_L_X6Y350.RAMB18_Y1.INIT_00[255:0] = 256'b0000...
BRAM_L_X6Y350.RAMB18_Y1.INITP_00[255:0] = 256'b0000...
```

So patching a design means rewriting those lines and reassembling.

## How it works

The openXC7 flow is:

```text
yosys           design.v   -> design.json
nextpnr-xilinx  design.json -> design.fasm     slow, placement and routing
fpga-as         design.fasm -> design.bit      fast
```

Only the memory initialisation changes when firmware changes. `xcbrup` takes

* the routed `design.fasm`,
* the `design.json` it was routed from,
* a fresh `design.json` built from the new memory contents,

and writes a new FASM with the new init bits spliced in. Then `fpga-as` turns
that into a bitstream. No placement, no routing.

To splice correctly it has to know which physical tile holds which memory. It
answers that question by content. Every RAMB18 half in a design carries 18432
init bits, and `xcbrup` computes the same 18432 bits from each cell in the JSON
and matches them against every half it finds in the FASM. The comparison covers
the entire memory, so a match is not a coincidence.

Two things follow from matching on content:

* Memories with different contents are located automatically. Nothing has to be
  wired up, annotated, or passed on the command line.
* Memories with *identical* contents cannot be told apart. That includes the
  common case of a memory that starts as all zeros, which is every runtime RAM
  in every design. `xcbrup` reports those as ambiguous and refuses to guess,
  because writing firmware into the wrong tile produces a bitstream that
  configures, reports done, and misbehaves.

If you have several memories that must be updatable and they would otherwise
hold the same data, initialise them with distinct filler in the routed build.
`xcbrup genpat` writes such filler. That is the same trick `brup` uses, for the
same reason.

## Requirements

* Python 3. No third-party modules. `xcbrup` is a single executable script; put
  it on your `PATH` or call it by path.
* The openXC7 toolchain, which provides `yosys`, `nextpnr-xilinx`, `fpga-as`,
  `openFPGALoader` and a matching prjxray database. The `toolchain-nix` flake is
  the easiest way to get a consistent set.

```bash
cd toolchain-nix
nix develop
```

Everything below assumes commands run inside that shell. In particular
`$PRJXRAY_DB_DIR` is set there, which `fpga-as` needs.

The tutorial uses designs from the `demo-projects` repository, which is a
separate checkout. Clone both and run the commands from the design directory.

Do not mix toolchain versions between the routed build and an update. If yosys
changes, the netlist changes, placement changes, and the routed FASM no longer
matches the new design. Re-route once after a toolchain upgrade.

## Tutorial: update firmware on an Arty S7

The `picosoc` demo is a good subject. It is a PicoRV32 softcore whose program
lives in `progmem.v` as an `initial` block, which yosys turns into block RAM.

### 1. Build once, for real

```bash
cd demo-projects/picosoc
make BOARD=arty_s7 picosoc.bit
```

This runs yosys, nextpnr-xilinx and fpga-as. Expect roughly 11 seconds of
synthesis and tens of seconds of place and route.

Check the board before flashing:

```bash
openFPGALoader --cable digilent --detect
```

You should see `model xc7s50`, `family spartan7`, `irlength 6`.

Flash and confirm the design runs:

```bash
openFPGALoader --cable digilent --bitstream picosoc.bit
stty -F /dev/ttyUSB2 460800 raw -echo
timeout 3 cat /dev/ttyUSB2
```

The board prints `Press ENTER to continue...`. Configuration alone proves
nothing; a design can load, report `done 1`, and sit there doing nothing. The
serial output is the evidence that the CPU is running.

### 2. Keep the routed pair

The FASM and the JSON must stay together. The JSON is the record of what the
router saw, and `xcbrup` needs it to recognise cells later.

```bash
cp picosoc.json picosoc-routed.json
cp picosoc.fasm picosoc-routed.fasm
```

Treat both as precious. `make clean` deletes them and you would have to route
again.

### 3. Change the firmware

Edit `progmem.v`. For this walkthrough, change one character of the greeting:
find the word holding `Pres`

```verilog
mem['h02B3] <= 32'h73657250;
```

and turn `P` (0x50) into `Z` (0x5A), keeping the bytes in the same order:

```verilog
mem['h02B3] <= 32'h7365725A;
```

### 4. Produce the new memory contents

**picosoc needs a synthesis run here**, and the reason is worth knowing. Its
`progmem` is inferred at a 36-bit port width, and the hex-only path below
currently reproduces only the narrower packing. `xcbrup learn` says so instead of
risking a wrong map:

```text
xcbrup: soc.progmem.mem: progmem.hex is 1024 x 32 bits, which needs 4 RAMB18E1
cells, but the routed design has 2
```

So, for this design: synthesis, no routing.

```bash
yosys -p "synth_xilinx -flatten -abc9 -arch xc7 -top top; write_json new.json" \
      arty_s7.v picosoc_noflash.v picorv32.v progmem.v simpleuart.v
```

### 4b. Or skip synthesis, where the layout is understood

When `learn` passes, no synthesis is needed at all. Run it once per routed
design; it rebuilds every init bit from the hex file and compares against the
JSON the router saw, so it also tells you which of the two paths you are on.

`demo-projects/bramtest` is such a design, and both of its memories are
verified:

```bash
xcbrup learn --json bramtest-routed.json \
    --mem mem0=bramtest.hex --mem mem36=bramtest36.hex
```

```text
  mem0: 15 x RAMB18E1, 8192 x 32 bits, 2048 rows per cell
  mem36: 2 x RAMB36E1, 4096 x 18 bits, 4096 rows per cell
PASS layout model reproduces every init bit of 2 memory(s)
```

Then patching is hex in, FASM out:

```bash
xcbrup patch --fasm bramtest-routed.fasm --json bramtest-routed.json \
    --mem mem0=bramtest.hex --mem mem36=bramtest36.hex \
    --new-mem mem0=new0.hex --new-mem mem36=new36.hex --output new.fasm
```

The `NAME` in `--mem NAME=FILE` is the memory's hierarchical name, which is the
cell name minus the last two components: `soc.progmem.mem.0.1` belongs to
`soc.progmem.mem`, `mem0.0.3` belongs to `mem0`.

### 5. Locate, assemble, flash

Worth doing once: confirm the memory can be found in the routed design.

```bash
xcbrup identify --fasm picosoc-routed.fasm --json picosoc-routed.json
```

```text
soc.progmem.mem.0.0 -> BRAM_L_X6Y95.RAMB18_Y0
soc.progmem.mem.0.1 -> BRAM_L_X6Y95.RAMB18_Y1
# matched 2/2 cells, 0 ambiguous, 0 unmatched
```

Both halves of the program landed in one tile. Your tile coordinates will differ;
the placer picks its own.

Either path ends the same way:

```bash
xcbrup update \
    --fasm picosoc-routed.fasm \
    --old-json picosoc-routed.json \
    --new-json new.json \
    --output new.fasm

fpga-as --prjxray_db_path=$PRJXRAY_DB_DIR/spartan7 --part xc7s50csga324-1 \
        new.fasm > new.bit

openFPGALoader --cable digilent --bitstream new.bit
```

Serial output now says `Zress ENTER to continue...`.

From editing firmware to a running board:

| step | time | path |
| --- | --- | --- |
| yosys | ~11 s | 4a only |
| `xcbrup update` | ~0.3 s | 4a |
| `xcbrup patch` | ~0.3 s | 4b |
| `xcbrup fit` | ~0.7 s, once per geometry | 4b |
| `fpga-as` | ~0.4 s | both |
| flash | ~3 s | both |

Place and route is gone either way, and on path 4b so is synthesis. The saving
grows with design size, since routing and synthesis both scale with the design
while patching scales with the size of the FASM file.

### 6. Repeat

Every later firmware change repeats step 3, then step 4, then step 5.
`picosoc-routed.fasm` and `picosoc-routed.json` stay untouched, and every new
bitstream derives from them.

## Changing memory contents without yosys

The loop in the tutorial re-runs yosys, which is only there to repack a hex file
into RAMB `INIT_xx` parameters. On a large design that synthesis is the remaining
wait, and it is avoidable: the repacking is a fixed layout that `xcbrup` can
reproduce itself.

Instead of a new JSON, hand `patch` a hex file of the new contents:

```bash
xcbrup learn --json picosoc-routed.json --mem mem=routed.hex
xcbrup patch --fasm picosoc-routed.fasm --json picosoc-routed.json \
    --mem mem=routed.hex --new-mem mem=new.hex --output new.fasm
```

`learn` proves the layout model against the routed design by rebuilding every
init bit from the hex file and comparing with the JSON. It exits non-zero if a
single bit disagrees, so a design whose layout is not understood is rejected
rather than patched on a guess. `patch` runs the same check before writing.

The model, in case you need to trust it:

* a plane is one bit of the word across a block of addresses, and blocks are
  sized by the cell depth (2048 addresses for a RAMB18, 4096 for a RAMB36);
* each RAMB cell holds nine consecutive planes as its nine columns, so a cell can
  straddle two address blocks and mix two bit ranges in one row;
* inside a cell, `INIT_xx[i*8+j]` is data bit `j` of row `xx*32+i`, and
  `INITP_yy[i]` is bit 8 of row `yy*256+i`.

`--mem NAME=FILE` takes the contents the routed design was built from, so the
argument is `NAME`, the module id yosys gave the memory: `mem0.0.3` belongs to
`mem0`. Use `xcbrup learn` first; if it passes, both hex files are interchangeable
with the JSON path.

Verified on the `bramtest` design: hex-only patching produces a FASM byte
identical to the yosys path, which is byte identical to a full nextpnr route of
the same contents.

Where the plane model stops: libmap picks a port width per memory, and this
model reproduces the packing for the narrow options. At width 36 the packing is
a spill rather than a plane stream. A `RAMB18E1` then holds 512 addresses of 36
slots each, but only 32 slots per address carry data, so the four spare slots of
the lower half take the upper half's low four bits and its remaining 28 bits go
to the second cell:

```text
cell .0.0   addr   0..511  bits 0..31  ->  slot 36*a + b
            addr 512..1023  bits 0..3   ->  slot 36*(a-512) + 32 + b
cell .0.1   addr 512..1023  bits 4..31  ->  slot 36*(a-512) + (b-4)
```

`slot = row * 9 + column`, and column 8 lives in `INITP`. Because the split
depends on how many words the initialiser sets, it is fitted for the design and
stored in a map file rather than derived from `depth` and `width`:

```bash
xcbrup fit --json picosoc.json --mem soc.progmem.mem=progmem.hex \
    --output maps/spartan7-ramb18-w36-1024x32.json

xcbrup patch --fasm picosoc.fasm --json picosoc.json --map maps/spartan7-ramb18-w36-1024x32.json \
    --mem soc.progmem.mem=progmem.hex \
    --new-mem soc.progmem.mem=new.hex --output new.fasm
```

A map is a list of per-cell segments of `(address, bit) -> slot`. `fit` does not
synthesise anything: it takes the routed JSON, which holds every init bit, and
the hex the design was built from, and solves for the few integers that describe
the packing, since a libmap packing is affine per segment. Picosoc takes 0.7 s.
The result is only written once it rebuilds the routed parameters bit for bit,
so a fit that does not fit is refused rather than saved.

`patch` repeats that check before writing anything, and refuses if a bit
disagrees, if the geometry does not match, or if the map was fitted for another
memory. A wrong or stale map cannot corrupt a bitstream.

Maps live in `maps/`, named by geometry rather than by design. The committed
`spartan7-ramb18-w36-1024x32.json` was also derived independently by 43
synthesis probes — 10 setting one address bit at a time, 32 of the form
`word[a] ^ (1 << j)` — and the two derivations agree on all 32,768 bits. The
fitting path is the one to use: it needs no toolchain beyond the files a build
already produced.

The map holds while the memory shape does. `progmem` therefore reads a
fixed-size hex file with `$readmemh`: an `initial` block whose length follows
the program would move the split on every edit and invalidate the map.

## Commands

### `genpat`

Write a pseudo-random hex file, for use as filler in a memory that would
otherwise collide with another one.

```bash
xcbrup genpat --output pattern.hex --width 32 --count 512 --seed 7
```

`--seed` distinguishes memories that have the same geometry.

### `cells`

List the RAMB cells of a yosys JSON with a digest of their init bits.

```bash
xcbrup cells --json picosoc-routed.json
```

```text
soc.progmem.mem.0.0    RAMB18E1   w=36 5b9f2571467871c1
soc.progmem.mem.0.1    RAMB18E1   w=36 10c5c42080bd79a4
```

### `halves`

List the RAMB18 halves of a FASM with the same digest.

```bash
xcbrup halves --fasm picosoc-routed.fasm
```

```text
BRAM_L_X6Y95    RAMB18_Y0 72 features 5b9f2571467871c1
BRAM_L_X6Y95    RAMB18_Y1 72 features 10c5c42080bd79a4
```

Equal digests are what a match looks like. These two commands exist to make that
visible when something will not match.

### `identify`

Match cells to FASM halves and report the mapping.

```bash
xcbrup identify --fasm design.fasm --json design.json [--map map.json]
```

Exits non-zero if any cell is unmatched or ambiguous. `--map` here writes the
cell-to-tile placement that was found; it is not the memory map `patch` takes.

### `update`

Splice the init bits of `--new-json` into `--fasm` and write `--output`.

```bash
xcbrup update --fasm design.fasm --old-json design.json \
              --new-json new.json --output new.fasm
```

Refuses, rather than producing a doubtful bitstream, when:

* a cell in the routed design cannot be located,
* the new design has a different number of RAMB cells,
* a cell changed type or port width,
* a memory is ambiguous and the new contents are not all zero.

Ambiguous cells whose new contents *are* all zero are skipped and listed. Writing
zeros over zeros changes nothing, so nothing is at risk. Runtime RAMs that start
zero fall into this category and do not need addressing.

### `learn`

Check that the block RAM layout model reproduces a routed design's init bits.
Reads the hex files the design was built from and compares, bit for bit, against
the JSON the router saw.

```bash
xcbrup learn --json design.json --mem mem0=mem0.hex --mem mem36=mem36.hex
```

```text
  mem0: 15 x RAMB18E1, 8192 x 32 bits, 2048 rows per cell
  mem36: 2 x RAMB36E1, 4096 x 18 bits, 4096 rows per cell
PASS layout model reproduces every init bit of 2 memory(s)
```

### `fit`

Fit a memory map from a routed design and the hex it was built from, for a
geometry the plane model cannot describe. No synthesis is run.

```bash
xcbrup fit --json design.json --mem mem0=mem0.hex --output maps/name.json
```

```text
  mem0.0.0: 512 addr x 32 bits at stride 36 base 0, 512 addr x 4 bits at stride 36 base 32
  mem0.0.1: 512 addr x 28 bits at stride 36 base 0
fitted map for mem0 (1024 x 32 bits, 2 RAMB18E1) -> maps/name.json
```

`--memory NAME` selects one memory when the design has several. The map is only
written once it rebuilds every init bit of the routed JSON; a packing outside
the family `fit` searches is refused, and the message says so. For widths up to
18 the plane model already applies, so `fit` is not needed there.

### `patch`

Rewrite memories in a routed FASM from hex files, with no synthesis.

```bash
xcbrup patch --fasm design.fasm --json design.json \
    --mem mem0=old0.hex --new-mem mem0=new0.hex --output new.fasm
```

`--map FILE` selects a fitted map for a geometry the plane model cannot
describe, such as a memory inferred at width 36. The map is checked against the
routed JSON first, and the run is refused if any init bit disagrees, if the
geometry differs, or if the map was fitted for a different memory.

Refuses when the layout check fails, when a hex file needs a different number of
cells than the routed design has, or when a memory cannot be located
unambiguously.

### `check`

Rewrite a FASM unchanged. A parser self-test; the output must be byte-identical
to the input.

```bash
xcbrup check --fasm design.fasm --output /tmp/same.fasm --cleanup
```

## What this does not do

* **Only block RAM.** `RAMB18E1` and `RAMB36E1` cells. Distributed RAM, LUT RAM
  and SRLs are not patched. If yosys implements your memory in a `RAM64X1D` or
  similar, `identify` reports nothing and there is nothing to update. Make the
  memory large enough, or force block RAM.
* **No runtime writes.** Only initial contents. If firmware reads and writes the
  same array, the patched contents are the power-on contents.
* **No structural change.** Memory size, count and port widths must stay fixed.
  Change those and you need a full run.
* **No constraint change.** Pin assignments, clocking, anything outside the
  memory init: full run.

## Verifying a build

A patched bitstream should be identical to one from a full run. Check that when
you upgrade the toolchain, or when you are unsure whether the routed pair still
corresponds to your design:

```bash
yosys ... write_json b.json           # firmware B, full synthesis
nextpnr-xilinx --chipdb ... --json b.json --fasm b.fasm
xcbrup update --fasm a.fasm --old-json a.json --new-json b.json --output p.fasm
cmp p.fasm b.fasm
```

If `cmp` is silent, patching is exactly equivalent to routing again. Differences
in lines other than `INIT_*` / `INITP_*` mean the two builds are not comparable,
usually because they came from different yosys versions.

Two things that look like problems and are not:

* `.bit` files differ by one byte at offset 97. `fpga-as` stamps the current time
  into the header. Compare FASM files, or compare the streams after the header.
* FASM files differ in a `# placer seed 12637216492877145455`-style comment. That
  line records the router's internal random state at the moment of writing
  (`nextpnr/himbaechel/uarch/xilinx/fasm.cc`). Equal values mean the two builds
  placed identically. A different value means a different netlist, not a
  different `--seed` argument.

## Current limits of the testing

Tested on `xc7s50` (Arty S7) with three designs:

* `demo-projects/picosoc`: 2 x `RAMB18E1` at width 36 holding a softcore program,
  patched through a fitted map. The map rebuilds all 32,768 initialised bits
  of the routed design; patching with unchanged firmware reproduces the routed
  FASM byte for byte; an edited firmware gives contents identical to a yosys
  synthesis of the same edit, matched by content across both cells; and the
  flashed bitstream printed the edited string with the original string absent.
* `demo-projects/bramtest`: 15 x `RAMB18E1` at width 9 holding 8192 x 32 bits, and
  2 x `RAMB36E1` at width 18 holding 4096 x 18 bits. Both memories are streamed
  over UART and compared word by word against a xorshift32 stream rebuilt from
  the seed, so 335872 memory bits are checked per bitstream.

  The hex-only path was also checked on hardware with a seed that no synthesis run
  had ever produced. Nothing but the layout model could have put the right values
  in those bits, and the board read all 335872 of them back correctly.

`bramtest` covers the `RAMB36E1` even/odd half interleave, which is the branch a
softcore-sized ROM never reaches, and the width-9 parity lanes, which carry data
in that configuration.

Each of those bitstreams was also checked against a full route of the same
design: the FASM from `xcbrup update` is byte identical to the routed FASM, and
the only lines that differ between two seeds are `INIT_*` and `INITP_*`.

Not tested: runtime write-back (only power-on contents are patched), persistence
from configuration flash, or any other 7-series family. Everything here is loaded
over JTAG into SRAM, so a power cycle clears the design and the board has to be
programmed again; nothing in that is specific to patching. Treat other families
as untested until you run the equivalence check on one of their designs.
