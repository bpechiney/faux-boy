# Mapper / Cart Review Checklist

Covers NES mappers (MMC1/2/3/5, UxROM, AxROM, MMC6, NROM, CNROM, etc.), Game Boy MBCs (none, MBC1/2/3/5/6/7, HuC1/3, MMM01, M161), and SNES enhancement chips (SA-1, SuperFX, S-DD1, S-RTC, DSP-1/2/3/4, Cx4, OBC1, ST010/11/18, SPC7110). Written with NES MMC3 as the running example because it exercises every methodology rule; `[GB]` and `[SNES]` deltas inline.

The cart is where compatibility breaks happen most often, because every bank-switch register interacts with the bus model. Read this checklist *and* re-read `bus_review.md` — they overlap deliberately.

---

## 1. IRQ tick granularity (methodology rule 2)

**MMC3-style mappers: IRQ counter clocked on PPU A12 *rising edges*, not CPU cycles?**
A12 toggles when the PPU fetches sprite tiles at $1xxx after BG fetches at $0xxx (or vice-versa, depending on `PPUCTRL.b` and `PPUCTRL.s`). Counting CPU cycles instead of A12 edges is the most common MMC3 bug — game IRQs fire on the wrong scanline, status bar wobbles or splits at the wrong row.
*Catches it:* Blargg `mmc3_test/`, especially `1-clocking.nes`, `2-details.nes`, `3-A12_clocking.nes`.

**A12 hardware filter implemented (debounce ~10–16 CPU cycles between counted edges)?**
Without the filter the counter clocks 3× too fast on every BG fetch and games fire IRQs every few dots. With the wrong filter window, edge-case CHR layouts misfire.
*Catches it:* `mmc3_test/4-scanline_timing.nes`, `5-MMC3_rev_A.nes`, `6-MMC3_rev_B.nes`.

**MMC3 rev A vs rev B distinction implemented (or explicitly chosen + commented)?**
Rev A reloads on counter==0 and decrements next; rev B reloads on the cycle the counter would have decremented from 0. Commercial games are split. Pick one with a comment, or detect from cart header.

**MMC5: scanline counter clocked from the PPU rendering signal, not A12?**
MMC5 is *not* MMC3 with a different banking layout. It uses its own internal scanline detection. Reviewing MMC5 with MMC3 mental model is a category error.

**`[GB]`** MBC3 RTC ticks at 1 Hz from the cart, independent of CPU clock. A "1 Hz = 4194304 CPU cycles" hardcode breaks under speed-up / TAS / save-state. The RTC is its own clock domain and should be modelled as such.

**`[GB]`** MBC5 has no IRQ. MBC7 (Kirby's Tilt 'n' Tumble) has the accelerometer but also no IRQ. If the cart has IRQ logic that isn't MBC3-RTC-related, suspect the mapper detection.

**`[SNES]`** SA-1 has its own 65C816 running at ~10.74 MHz with its own IRQ system, message register IRQs from main CPU to SA-1 and vice versa, and a configurable timer. Tick granularity is SA-1-cycle-level.

**`[SNES]`** SuperFX (GSU-1 / GSU-2) is a separate processor; its "IRQ" is really a halt-and-resume signal back to the 65C816. Cycle accounting must respect the cache prefetch — the GSU stalls the bus while filling its 512-byte cache from ROM.

---

## 2. Bank-switch on the last cycle (methodology rules 2 & 5)

**Writes to bank registers take effect *after* the bus cycle they occur on — so the *next* instruction byte may still come from the old bank?**
On real silicon, the bank select propagates after the data bus has been latched. Code that does `STA $8000 ; <next byte>` may have `<next byte>` fetched from the old bank if the addressing happens to land in the switched window. Most carts pad with safe NOPs or JMP through fixed banks; an emulator that switches "instantly" passes most games but breaks any that rely on the timing.

**The fixed bank window is *actually* fixed for the mapper variant?**
MMC1 has multiple PRG modes. UxROM fixes the upper 16 KiB. AxROM has no fixed window. CNROM has fixed PRG. Getting the wrong window fixed is silent compatibility loss — the test ROM list won't catch it.

**`[GB]`** MBC1 mode 0 vs mode 1 affects bank routing differently for large-RAM carts versus large-ROM carts; the per-mode register decode is documented at [pandocs — MBCs](https://gbdev.io/pandocs/MBCs.html). The review heuristic: a single-mode MBC1 implementation passes most carts but breaks the multi-megabyte and large-RAM ones in non-obvious ways. Mooneye `acceptance/mbc1/` covers the mode bit comprehensively.

**`[GB]`** MBC1 has a famous quirk: bank 0 selected in the $4000–$7FFF window reads as bank 1 (similarly 0x20, 0x40, 0x60). Not a bug — silicon behaviour, documented in [pandocs — MBCs](https://gbdev.io/pandocs/MBCs.html). Some games depend on it.

**`[SNES]`** ExHiROM, HiROM, LoROM mappings are bus-level memory maps, not bank registers. Pick the mapping from the cart header (with the documented header-detection heuristics for the 1% of carts with wrong/ambiguous headers).

---

## 3. Nametable mirroring (methodology rule 3)

**All four mirroring modes implemented: horizontal, vertical, single-screen low (NT0), single-screen high (NT1), four-screen (cart-supplied VRAM)?**
"We only support horizontal and vertical" leaves AxROM and others broken. Single-screen-with-write-bit-selecting-screen (MMC1, AxROM) is its own mode.

**Mirroring mode can change *mid-frame* on most mappers (MMC1, MMC3, MMC5)?**
Games like *Crystalis* change mirroring per scanline for split-screen. If mirroring is sampled once per frame, raster effects break.

**Four-screen mode uses cart-supplied 2 KiB VRAM as the second pair of nametables, not the console's 2 KiB mirrored differently?**
*Gauntlet* and *Rad Racer 2* depend on this. The extra VRAM is part of save-state.

**`[GB]`** No nametable concept; window WX/WY position is the equivalent runtime-mutable layout state.

**`[SNES]`** PPU nametable layout is per-BG via $2107–$210A; cart doesn't control mirroring. Mirroring quirks are in the PPU, not the mapper.

---

## 4. CHR-ROM vs CHR-RAM writes (methodology rule 3)

**CHR-RAM carts (CHR=0 in iNES header for relevant mappers): writes to $0000–$1FFF actually mutate the pattern table?**
Many games use CHR-RAM and DMA tile data from CPU RAM each frame. "Silently dropping CHR writes because we treat CHR as ROM" breaks any CHR-RAM title (Metroid, Final Fantasy, lots of homebrew).

**CHR-ROM carts: writes to $0000–$1FFF are silently dropped, NOT panicked?**
A poorly-coded game that accidentally writes to CHR-ROM should run, not crash the emulator.

**CHR window granularity correct for the mapper?**
MMC3 has 2 KiB and 1 KiB windows controllable via the chr-mode bit; flipping the bit swaps which windows are which. MMC5 has 8/4/2/1 KiB modes per BG vs sprite. Wrong window granularity = wrong tiles in unexpected places.

**`[GB]`** Cart RAM behaviour:
- RAM-enable register (writing 0x0A to $0000–$1FFF enables, anything else disables).
- Writes to RAM with RAM disabled: silently dropped, NOT panicked.
- Reads from RAM with RAM disabled: return 0xFF (or open bus on some MBC implementations) — *not* the underlying RAM value.
- Mooneye `acceptance/mbc1/ram_*` and the RAM-enable corner cases are the test surface.

**`[SNES]`** SRAM is on the cart; size in header. Some carts have SRAM mapped via DSP companion chip, not directly. PSRAM (battery-backed) vs scratchpad RAM (volatile) distinction matters for save-format correctness.

---

## 5. Implicit cart state (methodology rule 5)

**Walk every entry below**; verify each round-trips through the serializer/deserializer with no truncation, no host-endian assumption, and the correct version gate. This section is mechanical, not interrogative — the failure mode is omission, so the only correct review activity is enumeration. If a field on this list isn't in the serializer, it's a finding.

State that must round-trip through save/load even though it isn't an obvious "named field":

**NES MMC3:**
- Bank-select register ($8000 low 3 bits) — selects which of R0–R7 the next $8001 write hits.
- All eight bank registers R0–R7.
- IRQ latch (the value $C000 last loaded), IRQ counter (current value), IRQ reload pending flag, IRQ enable flag.
- Last-A12-edge timestamp (for the filter).
- Mirroring mode bit, PRG-mode bit, CHR-mode bit.
- WRAM enable + write-protect bits ($A001).

**NES MMC5:**
- Multiplier registers ($5205/$5206) and result.
- Fill-mode tile and color.
- Per-bank RAM-protect bits.
- ExRAM contents (1 KiB) and mode.
- Scanline IRQ counter, in-frame flag.

**`[GB]` MBC3:**
- RTC registers (S/M/H/DL/DH).
- RTC latch state — `00 → 01` write to $6000 latches the running time into the visible registers.
- RTC running-vs-halted flag (DH bit 6).
- Sub-second accumulator (the "what fraction of the next second has elapsed" counter).
- Last-real-time wall-clock anchor if the implementation drifts back to host time on load.

**`[SNES]` SA-1:**
- SA-1 CPU registers (PB/PC, A, X, Y, SP, DP, DBR, P, M, X flags, E flag, M/X widths).
- SA-1 vs main CPU IRQ control & message registers ($2200–$220B and $2300+).
- BWRAM mapping & bitmap mode state.
- SA-1 timer.
- DMA channel state (independent of S-CPU DMA).

**`[SNES]` SuperFX:**
- All 16 GSU registers (R0–R15).
- Status flags (Z, S, CY, OV, ALT1/ALT2, IRQ, RON, RAN, B).
- Cache contents (512 bytes) and cache-valid bits.
- Plot register, color register, screen-base register, pixel cache.
- ROM/RAM bus arbitration state.

A save-state that names only the bank registers and forgets the IRQ counter latch / RTC sub-second accumulator / GSU cache is broken; symptoms appear seconds-to-minutes after load.

---

## 6. Determinism (methodology rule 4)

**Cart RAM (SRAM, WRAM, expansion RAM) initialised to a deterministic pattern on first power-on (no save file present)?**
"Whatever was in the host's allocator" makes save files differ across runs. Conventions: all 0x00 (most common), all 0xFF, or per-cart override (some games depend on a known startup pattern).

**Save-file format is byte-exact, not host-endian-dependent?**
SRAM dumps must be portable across emulator hosts. `@bitCast` of a struct into the file is the trap.

**RTC time advance on load deterministic?**
If MBC3 RTC keeps running while the emulator is closed (by sampling host time on load), document it and offer a "frozen RTC" mode for TAS/regression. The two modes are both legitimate — just don't conflate them silently.

---

## 7. Cartridge / ROM loader, region & revision (methodology rule 7)

This section covers both region/revision review and the **cartridge / ROM loader** PR shape (header parsing, mapper detection from header bytes, checksum verification, fallback when header is wrong or ambiguous). The two concerns coincide because every region/revision-divergent decision is rooted in a header byte, and every header-parsing decision is the first place a wrong assumption silently locks in a wrong variant for the rest of the run.

Per-system header byte offsets and field decodings are standing-facts — do not inline them. Cite [nesdev wiki — iNES](https://www.nesdev.org/wiki/INES) and [NES 2.0](https://www.nesdev.org/wiki/NES_2.0), [pandocs — The Cartridge Header](https://gbdev.io/pandocs/The_Cartridge_Header.html), and [fullsnes — Cartridge Header](https://problemkaputt.de/fullsnes.htm#snescartridgeromheader) directly.

### Trust-but-verify: header bytes are advisory

**Header bytes are inputs to detection, not ground truth.**
The hardware doesn't read the header at all — the header is metadata for the loader. A header that disagrees with observable cart behaviour (file size, checksum, vector layout) means the header is wrong, not the cart. The loader must reconcile, not blindly trust.

**Header values cross-validated against file size?**
ROM-size and RAM-size bytes are *advisory*. Some pirate carts under-report; some legitimate carts under-report (NES 2.0 backfilled the iNES short-form for several titles). Trust the larger of (header-declared, file-size-implies). When they disagree by more than a small factor, log and fall back to file size.

**Mismatched checksum policy is explicit?**
Bad-checksum loads should not silently succeed *or* silently fail. Three valid postures: warn-and-load (most common), refuse (homebrew/release-engineering shape), or auto-overlay-with-database (No-Intro/GoodNES style). The bug-shape is silent acceptance with no log line — review for the log/warn path, not just the load path.

**Ambiguous-header fallback path documented?**
When the primary detection fails (e.g., SNES header location is ambiguous between $7FC0/$FFC0/$40FFC0), the fallback is a documented, deterministic heuristic — score each candidate location by checksum-plausibility + name-printability + region-byte-validity, pick the highest. The bug-shape is "first one that doesn't crash wins" or "always pick LoROM."

### Mapper detection from header bytes

**Mapper number derived from the header, not from the filename or extension?**
Filename-based mapper detection (e.g., reading "MMC3" from a directory name) is the bug-shape. The mapper number is in the header byte(s); detection that ignores them is silently picking the wrong variant for any rom that's been renamed.

**`[NES]` iNES vs NES 2.0 disambiguation: submapper byte and PRG/CHR-RAM-size byte parsed?**
NES 2.0 disambiguates several mappers (mapper 1 submapper 5 = SEROM/SHROM/SH1ROM; mapper 4 submappers for MMC3 rev A/B/C, MMC6, MC-ACC). Defaulting to "mapper N, no submapper info" silently picks the wrong variant. The detection rule: NES 2.0 magic in the right bytes → use submapper; otherwise → fall back to a documented per-mapper-number default (and log the choice).

**`[NES]` Trainer / four-screen flag / battery flag from header byte 6 / 7 routed to the right places?**
Trainer-present shifts the PRG offset by 512 bytes; missing this loads the trainer as the start of bank 0. Four-screen flag overrides mirroring; missing it produces wrong scrolling on *Gauntlet*-class carts. Battery flag drives whether to allocate persistent SRAM.

**`[GB]` Cart-type byte ($0147) maps to MBC variant *and* presence of RAM / battery / RTC / rumble?**
A `MBC1` card with type `0x03` (MBC1+RAM+BATTERY) needs persistent SRAM; type `0x01` (MBC1 only) does not. The loader must use the cart-type byte to drive both the mapper choice and the persistence-and-peripheral choices in one pass — splitting them produces drift (e.g., MBC chosen from one path, battery flag missed from another).

**`[GB]` Header checksum byte ($014D) and global checksum ($014E–$014F) verified, or explicitly skipped with a logged decision?**
Real DMG hardware verifies the header checksum on boot — a failing cart hangs at the boot logo. Emulators that skip the check pass *more* roms than real hardware (including some bad dumps); document whether your loader matches hardware (refuse) or is permissive (warn-and-load).

**`[SNES]` Header location ambiguity ($7FC0 LoROM vs $FFC0 HiROM vs $40FFC0 ExHiROM) resolved by scoring, not hardcoding?**
Standard heuristic per fullsnes: try each, score by checksum+complement consistency + name printability + map-mode byte plausibility, pick the highest. Hardcoding one location breaks the dual-header carts and any cart whose header layout doesn't match the filename/extension hint.

**`[SNES]` Map-mode byte ($xxFFD5) parsed for SlowROM vs FastROM, BS-X / SuperFX / SA-1 mapping flags?**
Map-mode is not just LoROM-or-HiROM — its low nibble encodes the variant. Misreading FastROM as SlowROM doesn't break the load but produces wrong CPU rate post-boot when the program writes to `MEMSEL`.

### Region & revision: threading vs hardcoding

**Region threaded as a parameter through every component that consumes it?**
The components: CPU divider, PPU scanline count, APU rate table, mapper-IRQ tick source, OAM-bug gate (GB), open-bus model (SNES 1-CHIP delta). The bug-shape is a hardcoded `const NTSC_DIVIDER = 12` somewhere downstream of the region decision; PAL roms then desync at audio rate without a code path that visibly says "PAL."

**`[NES]`** 2A03 (NTSC) vs 2A07 (PAL) divergence: master-clock divider (÷12 vs ÷16), DMC rate table, noise period table, frame-counter step boundaries. Hardcoding NTSC silently breaks PAL audio. Cite [nesdev wiki — Clock rate](https://www.nesdev.org/wiki/Cycle_reference_chart).

**`[GB]`** Hardware-revision-gated quirks: DMG OAM-bug presence (yes on DMG, no on CGB; both yes/no patterns are silicon-revision-correct), 1-CHIP DMG open-bus, AGB STAT-IRQ blocking edge case. The OAM bug specifically is *gated by silicon revision*, not universal. Cite [pandocs — Hardware revisions](https://gbdev.io/pandocs/Hardware_Reg_List.html).

**`[SNES]`** Original vs 1-CHIP open-bus delta, PAL vs NTSC frame timing (262 vs 312 lines + dot count), region-locked carts (header region byte vs console region). Hardcoding NTSC frame timing breaks PAL stress titles.

---

## 8. Mapper-bus interaction quirks (methodology rule 3)

**Bus conflicts on discrete-logic mappers (UxROM, CNROM, AxROM, BNROM)?**
These mappers have no internal latch — the bank register IS the data bus, ANDed with whatever the ROM is currently outputting. Writes that conflict produce ambiguous results. Most games avoid this by writing through RAM or by ensuring ROM data == intended bank value at the write address. Some games depend on the conflict resolving a specific way; emulator should at minimum not panic.

**Write to PRG-ROM region with no mapper register at the address: silently dropped?**
Different from "panic with unimplemented register". No mapper documentation lists every address; some games stray.

**Open bus from cart side (NROM reading $4020–$5FFF) returns CPU open-bus, not zero?**
Same rule as PPU/APU regions — see `bus_review.md`.

---

## 9. Citation hygiene (methodology rule 6)

For mappers, citations should name:

- **The mapper number and submapper** (e.g., "iNES 4.1 = MMC3 rev A").
- **The nesdev wiki page** (or pandocs page for MBCs, fullsnes for SNES enhancement chips).
- **A known-affected game** for non-obvious quirks (e.g., "Mega Man 3 splits one scanline late if A12 filter is < 8 cycles").
- **For Disch's docs:** acceptable as a citation, since they're the standard mapper-doc compendium.

Uncited mapper code is the bug. Mapper documentation is the densest external dependency in the entire emulator — losing the citation trail makes the next refactor unsafe.

---

## 10. Test-ROM correspondence (methodology rule 1)

This section maps **review triggers → ROMs to re-run**. For ROM identity follow the canonical-source pointers in `references/test_roms.md` to the upstream archive.

| Change touches... | Re-run at minimum (NES) |
|---|---|
| MMC3 IRQ / A12 filter | Blargg `mmc3_test/` (all 6) |
| MMC1 mode bits | bunch of MMC1 games — *Mega Man 2*, *Final Fantasy*; no canonical test |
| MMC5 | *Castlevania III*, *Just Breed*; MMC5 test ROMs (limited) |
| Mirroring mid-frame | *Crystalis*, *Battletoads* split-screen sections |
| CHR-RAM writes | *Metroid*, *Final Fantasy*; any CHR-RAM homebrew |
| Bank-switch timing | hard to isolate; suspect when an MMC3 or MMC1 game crashes inside its own routines |

| Change touches... | Re-run at minimum (Game Boy) |
|---|---|
| MBC1 banking & mode | mooneye `acceptance/mbc1/` (full directory) |
| MBC2 | mooneye `acceptance/mbc2/` |
| MBC5 | mooneye `acceptance/mbc5/` |
| MBC3 RTC | mooneye `misc/bits/unused_hwio-C.gb` (and cart-specific testing — RTC test ROMs are sparse) |
| RAM enable | every MBC mooneye test (it's a shared assumption) |

| Change touches... | Re-run at minimum (SNES) |
|---|---|
| LoROM/HiROM mapping | wide compatibility tests; no single ROM |
| SA-1 | *Super Mario RPG*, *Kirby Super Star*, *Kirby's Dream Land 3* — these are the SA-1 stress titles |
| SuperFX | *Star Fox*, *Yoshi's Island*, *Doom* (the GBA-port-of-the-SNES-port baseline) |
| S-DD1 | *Star Ocean*, *Street Fighter Alpha 2* (graphics decompression on-the-fly) |
| Cx4 | *Mega Man X2/X3* |
| SPC7110 | *Tengai Makyou Zero* |

A mapper change with no associated test path is a yellow flag — note it explicitly in the review output and propose the minimal homebrew test ROM that would isolate the behaviour (often easier to write than to find for mapper-specific bugs).
