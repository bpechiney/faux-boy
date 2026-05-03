# faux-boy

A cycle-accurate Game Boy emulator, written in Zig.

> **Status: Early development.** No source code is in the tree yet. Everything
> below is design intent. The test ROM table will fill in as the
> implementation lands.

## About

faux-boy is a from-scratch Game Boy emulator written in Zig, built as a
craft/learning project. The goal is full hardware fidelity, not raw
performance — the kind of accuracy that lets mid-frame raster tricks and
sub-instruction timing exploits run the way they did on real silicon.

## Accuracy Goals

- **T-cycle accurate.** Every component (CPU, PPU, APU, Timer, DMA) advances
  per T-cycle (4.194304 MHz), interleaved correctly within a single
  instruction.
- **Pixel-FIFO PPU.** The PPU is modeled as a per-dot pixel FIFO — not a
  scanline renderer — so mid-scanline writes to LCDC, SCX/SCY, and palettes
  produce the correct visual output.
- **Test-ROM driven.** Accuracy is validated against published hardware test
  suites (see below), all run unattended in the harness.

## Hardware Scope

- **Phase 1 — DMG.** Original 1989 Game Boy. Primary target.
- **Phase 2 — CGB (stretch).** Game Boy Color, including double-speed mode,
  HDMA, and CGB palettes. Tackled once DMG is rock-solid.
- **Phase 3 — link-cable peripherals (stretch).** Full two-instance + networked
  link cable. Phase 1 ships a null-peer/loopback so games that probe the link
  don't soft-lock; the lockstep scheduler for real link comes later.
- Out of scope: SGB, GBA, pocket variants.

Roadmap detail lives in the project meta-issue (link TBD).

## Cartridge & Peripheral Support

Hardware fidelity doesn't stop at the SoC. The cart talks to the CPU through
a memory-bank controller (MBC), and a handful of carts ship extra silicon —
RTCs, rumble motors, tilt sensors. Plus the link port has its own ecosystem.

### Memory-Bank Controllers

| MBC | Caps | Special features | Representative games |
|---|---|---|---|
| None | 32KB ROM | Direct map, no banking | Tetris, Dr. Mario |
| MBC1 | ≤2MB ROM / 32KB RAM | Mode-bit quirk that re-uses upper-bank bits for ROM-or-RAM banking | Pokemon R/B, Super Mario Land |
| MBC2 | ≤256KB ROM | Built-in 512×4-bit RAM (nibbles, not bytes) | Mario Land 2: 6 Golden Coins |
| MBC3 | ≤2MB ROM / 32KB RAM | Optional **RTC** — battery-backed clock for day/night cycles | Pokemon Gold/Silver/Crystal, Harvest Moon GB |
| MBC5 | ≤8MB ROM / 128KB RAM | Optional **rumble motor**. Standard late-era controller — used in many DMG-compatible carts, not CGB-tied | Pokemon Pinball, Wario Land II, Donkey Kong Country |
| MBC7 | 2MB ROM | 256-byte EEPROM + 2-axis **tilt sensor** (ADXL202) | Kirby Tilt 'n' Tumble |

All Phase 1. HuC1 / HuC3 / MMM01 (rare third-party) and the Game Boy Camera
cartridge are out of scope.

### Console & Peripherals

- **Boot ROM execution.** The real Nintendo boot ROM runs at startup —
  scrolling logo, chime, register handoff at `$0100` — instead of faking the
  post-boot register state. Required for some Mooneye `boot_hwio` tests and
  for CGB titles that sniff handoff registers to pick a palette.
- **Game Boy Printer.** Receive-side serial protocol, decode the tile
  bitmap, write a PNG. Pokemon Yellow/Crystal Pokedex pages and stickers are
  golden-tested against reference output.
- **Link cable.** Phase 1 ships a null-peer/loopback — the SB/SC registers
  acknowledge cleanly so games that probe the link don't soft-lock.
  Cycle-locked two-instance and networked link are Phase 3.

## User Experience

### Phase 1

- **Header-scanned ROM library.** Point faux-boy at a directory; it parses
  cart headers (title, MBC, CGB flag, RAM size) and presents a sortable,
  searchable list with last-played tracking. Double-click to launch.
- Save states (named slots).
- Screenshots.
- Fast-forward + slow-motion.
- Configurable keyboard + gamepad.
- Custom DMG palettes (4-shade picker).

### Phase 2

- **Rewind.** Hold a button, scrub back through the last N seconds. Beyond
  user value, it's a forcing function: rewind only works if save-state
  serialization is bit-for-bit complete and deterministic — which is exactly
  what the snapshot, replay, and reference-trace bisection machinery rely on.
- **A/V recording.** PNG sequence / WAV / MP4 capture for demos and bug
  reports.

Out of scope: shaders, LCD-grid effects, border art, per-game config.

## Test ROM Compliance

| Suite | Coverage | Status |
|---|---|---|
| [Blargg](https://github.com/retrio/gb-test-roms) | CPU instructions, instr/mem timing, halt/OAM bugs, APU | — |
| [Mooneye Test Suite](https://github.com/Gekkio/mooneye-test-suite) | ~150 acceptance tests: instructions, interrupts, OAM DMA, PPU timing, MBC1/2/3/5 | — |
| [dmg-acid2](https://github.com/mattcurrie/dmg-acid2) | PPU rendering correctness (composite scene) | — |
| [Mealybug Tearoom](https://github.com/mattcurrie/mealybug-tearoom-tests) | Mid-scanline PPU register writes (the cycle-accurate PPU litmus test) | — |
| [SameSuite](https://github.com/LIJI32/SameSuite) | APU edge cases, CGB DMA, PPU/interrupt quirks | — |

`—` = not yet attempted • ✅ = passing • ❌ = failing

Pass/fail is detected in three ways depending on suite:

- **Serial port** (Blargg) — link-cable writes are scraped for `Passed`/`Failed`.
- **Magic breakpoint** (Mooneye, SameSuite, others) — `LD B, B` halts the harness; CPU register pattern indicates pass/fail.
- **Framebuffer diff** (Acid2, Mealybug) — output frame compared byte-for-byte against a reference PNG.

## Development

faux-boy is built for an AI-driven dev loop — text-first tracelogs,
deterministic state dumps, reference-trace bisection — over GUI debuggers.
See [DEVELOPMENT.md](DEVELOPMENT.md) for the full toolchain story,
engineering practices (fuzz testing, multi-platform CI), and how to build
and run faux-boy locally.

## References

- [Pan Docs](https://gbdev.io/pandocs/) — the community-maintained Game Boy hardware reference.
- [Cycle-Accurate Game Boy Docs](https://github.com/AntonioND/giibiiadvance/blob/master/docs/TCAGBD.pdf) — Antonio Niño Díaz's deep dive on cycle timing.
- [Game Boy: Complete Technical Reference](https://gekkio.fi/files/gb-docs/gbctr.pdf) — Gekkio's hardware breakdown.
- Test ROM suites — see the table above.

## Acknowledgments

faux-boy stands on the shoulders of the gbdev community. Particular thanks to
the authors and maintainers of Pan Docs, the Cycle-Accurate Game Boy Docs, and
the test ROM suites listed above — without those resources, accurate Game Boy
emulation outside Nintendo would not be tractable.

## License

GPL-3.0 — see [LICENSE](LICENSE).
