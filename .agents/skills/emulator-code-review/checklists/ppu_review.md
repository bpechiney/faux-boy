# PPU Review Checklist

Covers NES PPU (RP2C02/2C07), Game Boy PPU (DMG/CGB), and SNES PPU (S-PPU1/S-PPU2). Written with NES as the primary case; `[GB]` and `[SNES]` deltas inline.

Walk top-to-bottom. The PPU is where rule 3 (missing behaviour) bites hardest — most "the game looks wrong" bugs are documented quirks the emulator silently omits.

---

## 1. Scanline / dot accounting (methodology rule 2)

**Scanline & dot counts match the region?**
The per-region scanline counts, dot counts, and visible/vblank/pre-render boundaries are documented at [nesdev wiki — PPU rendering](https://www.nesdev.org/wiki/PPU_rendering) (NES NTSC/PAL/Dendy). Region must thread through, not be hardcoded — see §9.
*Catches it:* Blargg `ppu_vbl_nmi` (NTSC), `pal_apu_tests` for region split.

**Pre-render scanline (-1 / 261) odd-frame dot-skip implemented — and ONLY when background rendering is enabled?**
On odd frames, dot 339 of the pre-render scanline is skipped (line is 340 dots instead of 341), but only if BG rendering (`PPUMASK.b`) is on. Forgetting this drifts NTSC by half a dot per frame, breaking timing-sensitive demos.
*Catches it:* `ppu_vbl_nmi` test 9; visible in raster effects in Battletoads.

**`[GB]`** Per-scanline mode lengths (mode 2 / mode 3 / mode 0) are documented at [pandocs — Pixel FIFO](https://gbdev.io/pandocs/pixel_fifo.html) and the [GBEDG cycle-accurate docs](https://hacktix.github.io/GBEDG/). Mode 3 length is variable (sprite count, SCX, window) — **a constant-length mode 3 is a bug**, the single hardest part of GB-PPU review.

**`[SNES]`** Per-region scanline counts and the interlace / non-interlace field-2 line-count delta are at [fullsnes](https://problemkaputt.de/fullsnes.htm). Frame skip with HDMA disabled vs enabled differs — verify both paths.

---

## 2. VBlank flag and NMI race (methodology rules 2 & 4)

**`PPUSTATUS.vblank` ($2002 bit 7) is set at scanline 241 dot 1, cleared at pre-render dot 1?**
Off by even one dot is observable — see `ppu_vbl_nmi` tests 3, 5, 6, 7.

**`$2002` read at the *exact* dot vblank is set: returns vblank=0 AND suppresses the NMI for that frame?**
The two-cycle race window (scanline 241 dots 0–2) is a real silicon behaviour. Reading `$2002` in that window: clears the latch, returns vblank=0, and suppresses NMI. Many games rely on the *non*-suppression case; some demos test the suppression directly.
*Catches it:* `ppu_vbl_nmi` test 7 (`07-nmi_on_timing.nes`), `vbl_nmi_timing/`.

**NMI line follows the AND of `PPUSTATUS.vblank` and `PPUCTRL.nmi_enable`, not just the rising edge of vblank?**
Setting `PPUCTRL.nmi_enable` while vblank is already high must trigger NMI immediately (level-triggered into the CPU edge detector). Toggling `nmi_enable` off then on within vblank can fire multiple NMIs.
*Catches it:* `nmi_sync`, `vbl_nmi_timing/06-nmi_control.nes`.

**`[GB]`** STAT interrupt is **edge-triggered on the OR of all enabled sources**, with the famous "STAT IRQ blocking" quirk: if any source is already high when another goes high, no new IRQ fires. Mooneye `acceptance/ppu/stat_irq_blocking` confirms.

**`[SNES]`** NMITIMEN ($4200) gates NMI/IRQ; reading $4210 clears the NMI flag. HDMA fires during HBlank regardless of NMI state.

---

## 3. Sprite 0 hit and sprite overflow (methodology rules 2 & 3)

**Sprite 0 hit set at the dot the opaque sprite-0 pixel meets the opaque BG pixel — not at scanline start?**
The flag goes up *during rendering*, not after the line completes. Games (notably SMB1's status-bar split) sample `$2002` while polling and need the timing right to within a few dots.
*Catches it:* Blargg `sprite_hit_tests_2005.10.05/`, especially `09-timing.nes`, `10-timing_order.nes`.

**Sprite 0 hit suppressed during dots 1–8 if BG or sprite clipping is enabled (`$2001` bits 1–2 = 0)?**
Dots 0–7 are always suppressed; dots 1–8 if leftmost columns are masked. Easy to over- or under-suppress.

**Sprite-overflow flag (`PPUSTATUS` bit 5): hardware-buggy implementation, not "fixed"?**
Real silicon increments the wrong index when scanning OAM for >8 sprites, which produces both false positives and false negatives that *games tolerate*. **Do not "fix" this** — a "correct" implementation that flags overflow accurately breaks games. The hardware-bug behaviour is documented at [nesdev wiki — PPU sprite evaluation](https://www.nesdev.org/wiki/PPU_sprite_evaluation); cite it. The review red flag: a comment claiming the implementation matches hardware while the code looks like a clean "scan all 64 sprites" loop — silicon is *not* clean here, and a clean loop is the bug.
*Catches it:* `sprite_overflow_tests/03-timing.nes`, `04-obscure.nes`.

**`[GB]`** No sprite-0; instead the LY=LYC compare at line start (with the 1-dot delay between LY change and STAT update) is the equivalent gotcha. 10-sprites-per-line cap, with priority by OAM index (DMG) or X coordinate (CGB).

**`[SNES]`** Range over (>32 sprites/line) and time over (>34 sprite tile fetches/line) flags in $213E. Both must reflect actual rendering; many games depend on neither tripping.

---

## 4. PPUDATA ($2007) read buffer (methodology rules 3 & 5)

**Reads from $0000–$3EFF return the *previous* buffered value, then refill the buffer?**
The first read after pointing VRAM at a tile returns whatever was previously in the buffer (often stale palette or open-bus). Reading twice and discarding the first is the documented idiom — emulating the buffer wrong silently breaks any code that relies on this latency.

**Reads from $3F00–$3FFF (palette) return the value *immediately* AND refill the buffer with the underlying nametable byte at `address - 0x1000`?**
Two distinct semantics. Easy to forget the buffer refill from the mirrored nametable region.
*Catches it:* `ppu_read_buffer/`, Blargg.

**VRAM address auto-increments by `PPUCTRL.increment_mode` (1 or 32) on EVERY $2007 read or write — including reads that returned the buffer, not the new data?**
Increment on access, not on "useful" access.

**During rendering (visible scanlines or pre-render with rendering enabled), $2007 access does NOT do a normal increment — it triggers a coarse-X+Y increment of `v` instead?**
This corrupts the scroll registers, which is *correct* hardware behaviour. Games that hit this are buggy on real hardware too; emulator must match.

---

## 5. Address latch (`w` toggle) shared between $2005 and $2006 (methodology rule 5)

**`w` is a single toggle bit, NOT one per register?**
Writes to $2005 and $2006 share the latch. Sequence `$2006 hi → $2005 → $2006 lo` produces a defined-but-weird state because the second $2005 write goes into the "low byte" position. Cleared by reading $2002.
*Catches it:* `ppu_open_bus`, `vbl_nmi_timing`, basically anything that scrolls.

**Reading $2002 clears `w`?**
The most-tested behaviour in the entire NES. Almost every NMI handler depends on this.

**Save-state serialises `w`, the temporary `t` register (15-bit), the current `v` register (15-bit), and fine-X (3-bit) explicitly?**
These are not memory-mapped — they're internal latches and the most commonly forgotten save-state field after the OAM evaluation state.

---

## 6. OAM and OAMDATA quirks (methodology rules 3 & 5)

**Writes to $2004 (OAMDATA) during rendering corrupt OAM in the documented way?**
Writes during rendering increment OAMADDR by 4 (skipping bytes) and the data may not actually land. Games that write OAM while rendering is on are buggy on hardware; emulator should reproduce the brokenness, not silently fix it.

**OAMADDR is forced to 0 during cycles 257–320 of pre-render and visible scanlines (sprite-fetch phase)?**
Some games leave OAMADDR non-zero and depend on the forced-0 behaviour for correct sprite eval next frame.

**$4014 OAM DMA: 256 bytes copied via 256× alternating read/write cycles, CPU stalled for 513 or 514 cycles depending on whether the DMA started on an odd CPU cycle?**
Both the duration and the bus access pattern are observable (DMC fetch contention).
*Catches it:* `oam_stress`, `dma_2007_write`.

**`[GB]`** OAM DMA bus conflict: during DMA, CPU reads from non-HRAM return $FF (or open bus). Game must execute from HRAM. Mooneye `oam_dma/` suite covers it. Also DMG OAM corruption bug on `inc rr`/`dec rr`/`pop rr` when `rr` happens to point inside OAM during mode 2 — see [pandocs — OAM Corruption Bug](https://gbdev.io/pandocs/OAM_Corruption_Bug.html); the silicon-revision-gate heuristic lives in checklists/mapper_review.md §7.

**`[SNES]`** OAMADDR has high-priority sprite designation; OAMDATA writes are word-paired (low byte buffered, both written on high byte). Mid-frame OAM access mostly forbidden.

---

## 7. Background fetch pipeline (methodology rule 2)

**8-cycle BG fetch pattern: NT byte → AT byte → pattern low → pattern high, latched into shift registers at the *correct* dot?**
Off-by-one in the latch produces a one-tile horizontal scroll offset that "works" but breaks raster-effect games (mid-frame scroll changes show one tile wrong).

**`v` register coarse-X increment at dots 8, 16, …, 256, and Y increment at dot 256?**
Both wrap with the documented quirks (Y=29→0 with vertical-nametable flip; Y=30 or 31 doesn't flip but does wrap to 0).

**Horizontal-bits-of-v copied from `t` at dot 257 of every visible & pre-render scanline?**
Pre-render scanline additionally copies *all* of `t` to `v` between dots 280 and 304. Forgetting either breaks vertical scroll setup.

**`[GB]`** PPU mode 3 fetcher pipeline (BG fetch → push to FIFO; sprite fetch interrupts BG fetch). SCX & 5 controls the initial FIFO discard count for fine X scroll. Window fetcher resets internal X but uses its own line counter (`WLY`).

---

## 8. Determinism (methodology rule 4)

**Frame buffer is deterministic given identical CPU/PPU state — no host-RNG, no time-based dithering, no host-endian-dependent pixel packing?**
Golden-frame regression tests rely on this.

**Palette LUT is a fixed table (e.g., FBX, Blargg, Smooth) chosen at build/runtime — not host-dependent?**
Any choice is fine; leaving it implicit is the bug.

**Open-bus value used for unmapped PPU register reads is the *PPU* open-bus latch (with documented per-bit decay), not the CPU one?**
Mixing them up usually surfaces in test ROMs that read $2002 with bits 0–4 expected to come from the PPU latch.

**Open-bus decay claim matches code reality?**
Per-bit decay (~600 ms on real silicon) only matters for very long idle reads — most emulators ignore decay and that's fine. The review red flag is a comment that *claims* decay is implemented while the latch value never actually changes; either delete the comment or wire the decay through. Reference: [nesdev wiki — Open bus behavior](https://www.nesdev.org/wiki/Open_bus_behavior).

---

## 9. Region & revision (methodology rule 7)

**NES: NTSC vs PAL vs Dendy threaded through as a parameter, not hardcoded?**
Affects: scanline count, vblank start line, CPU/PPU clock ratio (NTSC vs PAL ratios at [nesdev wiki — Cycle reference chart](https://www.nesdev.org/wiki/Cycle_reference_chart)), APU rate tables, sprite-overflow timing margins. A hardcoded ratio silently desyncs whichever region didn't get the constant.

**`[GB]`** DMG vs MGB vs SGB vs CGB vs AGB: OAM corruption bug present only on DMG/MGB; window WX=0/166 quirks; CGB has FIFO behaviour differences and double-speed mode (CPU 2× but PPU 1×).

**`[SNES]`** Original vs 1-CHIP-01 vs 1-CHIP-02 vs Mini: video signal differences, not usually emulator-relevant; revision-specific PPU corner cases (e.g., mid-frame mode-7 changes) are.

---

## 10. Citation hygiene (methodology rule 6)

The PPU is the densest quirk surface in the entire system. Apply rule 6 strictly:

- **Cited to nesdev / pandocs / fullsnes / a named test ROM?** ✅ Trust it.
- **"It looked wrong without this" with no source?** ❌ This is the bug. Either the fix is right and needs a citation so it survives refactors, or it's wrong and the actual bug is elsewhere.
- **A comment that says "this matches real hardware" with no test ROM named?** ⚠️ Ask which one.

---

## 11. Test-ROM correspondence (methodology rule 1)

This section maps **review triggers → ROMs to re-run**. For ROM identity (what each ROM tests, result address, pass/fail interpretation) follow the canonical-source pointers in `references/test_roms.md` to the upstream archive.

| Change touches... | Re-run at minimum (NES) |
|---|---|
| VBlank flag, NMI control | `ppu_vbl_nmi/`, `vbl_nmi_timing/`, `nmi_sync` |
| Sprite 0 hit | `sprite_hit_tests_2005.10.05/` |
| Sprite overflow flag | `sprite_overflow_tests/` |
| $2007 read buffer | `ppu_read_buffer`, `vbl_nmi_timing` |
| OAMDATA / OAM DMA | `oam_stress`, `oam_read`, `dma_2007_write` |
| $2005/$2006 latch / scroll | any scrolling game; `scrolltest_ntsc` |
| Open bus | `ppu_open_bus` |
| Palette / rendering | `full_palette`, `color_test` |

| Change touches... | Re-run at minimum (Game Boy) |
|---|---|
| LCDC / STAT / mode timing | mooneye `acceptance/ppu/`, `acceptance/timer/` |
| OAM DMA | mooneye `acceptance/oam_dma/` |
| Window | mooneye `acceptance/ppu/wx_*`, dmg-acid2 |
| Sprite priority / 10-per-line | dmg-acid2, cgb-acid2 |
| LCD enable mid-frame | mooneye `acceptance/ppu/stat_lyc_onoff` |

| Change touches... | Re-run at minimum (SNES) |
|---|---|
| HDMA timing | peter_lemon `HDMA/`, pvsneslib HDMA tests |
| Sprite range/time over | peter_lemon `OAM/`; some libsfx tests |
| Mode 7 | peter_lemon `Mode7/` |
| H/V counters | peter_lemon `HV/` |

A PPU change with no associated test-ROM verification path is a yellow flag — note it explicitly in the review output and propose what new test would catch the bug.
