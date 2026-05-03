# Test ROM Reference

Per-ROM identity (what each ROM tests, where it writes results, how to interpret pass/fail) lives in the upstream archives — follow the canonical-source pointers at the bottom of this file. The checklists' "Test-ROM correspondence" sections route review-triggers to ROM names; identity lookup is a search against those upstream archives.

**What survives here:** the *meta*-rule that some hardware behaviours have **no canonical test ROM** at all. These are the gaps reviewers should not expect coverage for — flag explicitly when a finding falls into one of these holes.

---

## "No canonical test ROM" gaps

When a review touches one of these areas, do not ask "which test ROM catches it?" — there isn't one. The fallback strategy is named with each gap.

### NES APU

- **Mixer non-linearity.** Verified by listening test against reference recordings (NES audio captured from FCEUX-with-Blargg-mixer or real hardware capture). Cross-reference: `checklists/apu_review.md` §6 "Sample resampling."

### Game Boy APU

- **Comprehensive LFSR initialisation.** Listening test against reference; mooneye-gb does not exhaustively cover. Cross-reference: `checklists/apu_review.md` §5 "Channel 4 (noise) LFSR."
- **DMG CH3 wave RAM corruption bug.** Some mooneye-gb coverage; pandocs documents the silicon behaviour but real-game dependency is rare. Cross-reference: `checklists/apu_review.md` §5 "Channel 3 (wave) quirks."

### Game Boy Mapper

- **MBC3 RTC sub-second accumulator and latch sequence.** Cart-specific testing only; pandocs documents but no isolated ROM. Cross-reference: `checklists/mapper_review.md` §5 "MBC3" implicit-state walk.

### SNES Coprocessors

- **SA-1, SuperFX, S-DD1 generic timing.** Real-game stress titles only:
  - SA-1: *Super Mario RPG*, *Kirby Super Star*, *Kirby's Dream Land 3*, *Marvelous*.
  - SuperFX: *Star Fox*, *Yoshi's Island*, *Stunt Race FX*, *Doom*, *Vortex*.
  - S-DD1: *Star Ocean*, *Street Fighter Alpha 2*.
  - Cx4: *Mega Man X2*, *Mega Man X3*.
  - SPC7110: *Tengai Makyou Zero* (Far East of Eden Zero).

### SNES Bus

- **WRAM-targeting-DMA-during-CPU-WRAM-access hazard.** Verify against bsnes-emu/bsnes source comments and the byuu/Near technical notes. Cross-reference: `checklists/bus_review.md` §2 [SNES].

### Save state

- **No canonical test ROM** for save/load round-trip — this is emulator-internal test infrastructure, not a published ROM. The closest equivalent: any deterministic test ROM that produces a known framebuffer at cycle N can serve as a save-state round-trip target. Run to cycle N/2, save, restore, run to N, hash framebuffer — must match the no-save-load reference hash. Cross-reference: `checklists/save_state_review.md` §5.

---

## Canonical sources (where to obtain test ROMs)

When a checklist's "Test-ROM correspondence" table names a ROM, look it up here:

- **Blargg's NES tests:** mirrored on nesdev wiki; original at `blargg.parodius.com` (intermittently online). Covers `nestest`, `cpu_test-v5`, `cpu_dummy_reads`, `cpu_dummy_writes_*`, `cpu_interrupts_v2`, `instr_misc`, `instr_timing`, `branch_timing_tests`, `cpu_timing_test6`, `cpu_reset`, `nmi_sync`, `ppu_vbl_nmi`, `vbl_nmi_timing`, `sprite_hit_tests_2005.10.05`, `sprite_overflow_tests`, `ppu_read_buffer`, `ppu_open_bus`, `oam_stress`, `oam_read`, `dma_2007_write`, `apu_test`, `frame_irq_test`, `dmc_dma_during_read4`, `length_counter`, `square_timer_div2`, `apu_reset`, `mmc3_test` (1–6), `full_palette`, `color_test`.
- **Blargg's GB tests:** `cpu_instrs`, `instr_timing`. Mirrored on nesdev / GitHub clones of `mooneye-gb`.
- **mooneye-gb:** [`github.com/Gekkio/mooneye-gb`](https://github.com/Gekkio/mooneye-gb) — source + prebuilt ROMs. The `acceptance/` tree covers `halt_bug`, `ei_*`, `di_timing-GS`, `reti_intr_timing`, `oam_dma/`, `timer/`, `ppu/` (mode timing, LCDC, LYC, sprite-priority, intr_2_mode0_timing, stat_irq_blocking, stat_lyc_onoff, lcdon_timing-GS, wx_*), `mbc1/`, `mbc2/`, `mbc5/`, `sound/` (length_counter, NR52 master enable), `bits/unused_hwio-*`, `intr_*`, `misc/`.
- **dmg-acid2 / cgb-acid2:** [`github.com/mattcurrie/dmg-acid2`](https://github.com/mattcurrie/dmg-acid2), [`github.com/mattcurrie/cgb-acid2`](https://github.com/mattcurrie/cgb-acid2) — visual rendering correctness (sprite priority, BG/window interaction, CGB palette modes).
- **peter_lemon SNES tests:** [`github.com/PeterLemon/SNES`](https://github.com/PeterLemon/SNES) — `CPUTest`, `CPUMSXTest`, `CPUDirectPageTest`, `OAM`, `HV`, `Mode7Test`, `SPCTest`, `OpenBus`, `DMA`, `HDMA`.
- **bsnes validation:** [`github.com/bsnes-emu/bsnes`](https://github.com/bsnes-emu/bsnes) — source comments serve as documentation when isolated ROMs don't exist (replaces the now-404 `devinacker/bsnes-mercury`).

URLs may rot — verify the canonical project before quoting paths in code review output. The reference here is project-name-and-author, not URL.
