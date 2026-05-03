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

## Accuracy goals

- **T-cycle accurate.** Every component (CPU, PPU, APU, Timer, DMA) advances
  per T-cycle (4.194304 MHz), interleaved correctly within a single
  instruction.
- **Pixel-FIFO PPU.** The PPU is modeled as a per-dot pixel FIFO — not a
  scanline renderer — so mid-scanline writes to LCDC, SCX/SCY, and palettes
  produce the correct visual output.
- **Test-ROM driven.** Accuracy is validated against published hardware test
  suites (see below), all run unattended in the harness.

## Hardware scope

- **Phase 1 — DMG.** Original 1989 Game Boy. Primary target.
- **Phase 2 — CGB (stretch).** Game Boy Color, including double-speed mode,
  HDMA, and CGB palettes. Tackled once DMG is rock-solid.
- Out of scope: SGB, GBA, pocket variants.

Roadmap detail lives in the project meta-issue (link TBD).

## Test ROM compliance

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

## Build & run

faux-boy targets Zig **0.16.0**. The repo ships a Nix flake that pins the
toolchain and a `justfile` for the common verbs.

```sh
# Enter the pinned dev shell (Zig 0.16.0, just, gh, jq)
nix develop

# Inside the shell:
just build         # zig build
just run <rom>     # zig build run -- <rom>
just test          # zig build test
just check         # CI parity: build + run all tests inside the dev shell
just fmt           # zig fmt src/ build.zig
just fmt-check     # CI parity: verify formatting without rewriting
```

Without Nix: install Zig 0.16.0 manually and run `just …` (or `zig build …`)
directly.

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
