# Microsoft Band firmware archive

Firmware images for Microsoft Band devices, plus the results of unpacking and
decompiling them, kept here for reference and future analysis. 

## Layout

```
<Device>/<Firmware version>/
    <container>.bin           the original firmware, as downloaded
    split_firmware.py         unpacks the container into its sections
    sections/                 every section extracted from the container
    strings/                  UI string tables extracted from language sections
    disassembly/               ARM Thumb/Thumb-2 disassembly (Capstone, superseded by decompiled/)
    decompiled/                C pseudocode from Ghidra's decompiler + a function index
    ghidra/                     scripts and load parameters that reproduce the decompilation
    README.md                   findings specific to that device/version
```

Currently archived:

- [`Band 2/2.0.5202.0`](Band%202/2.0.5202.0) -- the last public Band 2 firmware.

## Decompilation strategy

1. **Unpack the container.** The `.bin` firmware Microsoft distributes is not
   a raw flash dump -- it's a proprietary container ("Envoy") holding a
   bootloader, a main application image, resource blobs, and one string table
   per supported UI language. This format was not reverse-engineered here:
   `split_firmware.py` is adapted from the `construct` schema documented in
   [msband](https://github.com/hire-marat/msband) (`src/msband/firmware.py`),
   released under **The Good Idea 2 License** -- reproduced verbatim in
   [`LICENSE-THE-GOOD-IDEA-2.txt`](LICENSE-THE-GOOD-IDEA-2.txt) per that
   license's own redistribution requirement.
2. **Identify load parameters for each executable section.** The container
   header's `Base` field is not always where a section actually executes from
   (see the RAM-relocation finding in the Band 2 2.0.5202.0 README) -- the
   claimed base has to be verified against the bytes by checking that the
   vector table it implies really does resolve to valid, in-range Thumb
   function pointers.
3. **Seed entry points before auto-analysis.** A custom Ghidra script
   (`ghidra/SeedVectorTable.java`) walks the Cortex-M vector table and marks
   every valid handler as a function/entry point before Ghidra's own
   auto-analysis runs, which measurably improves function discovery over
   letting Ghidra find code purely by linear sweep.
4. **Batch-decompile and export.** `ghidra/ExportDecompiled.java` runs
   Ghidra's decompiler over every discovered function and writes one `.c`
   file plus a CSV index (address, name, size, decompiled-ok) per section.
5. **Verify claims against evidence, not assumption.** Every non-obvious
   finding in a version's README (RAM-relocation, cross-section calls,
   recompilability) was confirmed by testing the hypothesis against the
   actual bytes or by attempting the operation, not by inference alone -- see
   each version's README for the specifics.

## Findings so far

- Some sections are linked to run from RAM even though the container stores
  them at a flash address -- loading at the header's `Base` silently produces
  a much lower function count and broken cross-references.
- The main application image calls into the (resident) bootloader image at
  runtime; the bootloader isn't only a boot-time loader, it acts as a shared
  service layer.
- Per-section integrity is a plain CRC-32, not a cryptographic signature.
- See each version's own README for the full, version-specific writeup.

## Can this be recompiled into new firmware?

No. What Ghidra produces is pseudocode reconstructed from machine code, not
the vendor's source -- it doesn't compile as-is (undeclared types, unresolved
data references, no recovered struct layout), and a rebuildable source tree
would additionally need the hardware register map, the RTOS, a matching
linker script, and a toolchain whose codegen matches the original exactly.
What this output is good for: understanding what the firmware does, and
locating specific behavior precisely enough for small, surgical binary
patches -- which remains untested against real hardware and could brick a
device with no documented recovery path.

## Contributing

Please don't include anything besides firmware binaries and their derived
analysis here -- no device-specific personal data, pairing keys, or anything
extracted from a live device rather than the firmware image itself.
