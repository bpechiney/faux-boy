# Development

This document covers the developer experience of Faux Boy: the AI-driven dev
loop the project is built around, plus how to build and run it locally. See
[README.md](README.md) for the project overview and feature scope.

## Built for an AI-Driven Dev Loop

Faux Boy is being built *with* an AI agent. That inverts the usual emulator
tooling priorities — text > GUI, deterministic > interactive, diff-friendly >
visually pretty. The dev loop is **change → run a deterministic test → diff
text output → spot the divergence → fix.** Everything below serves that loop.

- **Headless, deterministic CLI.** Same input + same ROM = byte-identical
  output. No SDL window, no audio device, no walltime randomness in headless
  mode.
- **Structured state dumps.** JSON of CPU / PPU / APU / timer / memory at
  any T-cycle boundary.
- **T-cycle tracelog.** Every executed instruction with cycle counter, PC,
  opcode mnemonic, registers, and memory reads/writes — stable text,
  greppable, bisectable.
- **Reference-trace bisection.** Matching tracelog format with SameBoy and
  Gambatte; binary-search for the first divergent T-cycle. Converts "the
  screen looks wrong" into "DIV increments at T-cycle 4194 instead of 4196."
- **Structured JSON test-ROM harness output.** Not just `PASS` / `FAIL` — a
  per-test record with expected vs actual register state, so iteration loops
  can read it directly.
- **Input record + replay.** Capture button-press timelines, replay
  deterministically. Turns one-off bugs into permanent regression tests.
- **Snapshot / golden tests.** Load state, run N cycles, assert state matches
  a frozen snapshot. Failures are diff-readable.
- **Symbolic disassembly + `.sym` files.** Tracelogs show
  `Main:gameLoop+0x12` instead of `0x0238`.
- **Scriptable breakpoints.** `--break-at PC=0x1234 --then dump-state` —
  CLI-driven, composable in shell pipelines.

Explicitly **out of scope**: GUI debugger, visual VRAM/sprite/palette
viewers, audio scope, cheat injection (GameShark/Game Genie). They cost
engineering time without serving the AI dev loop.

## Engineering

- **Fuzz testing.** Two flavors: random-byte ROM fuzz (generate garbage cart
  data, run for N cycles, assert no crash) and random-input fuzz on real
  ROMs (mash buttons to surface PPU/timer edge cases no test ROM hits).
- **Multi-platform CI.** GitHub Actions running the build + the full test
  ROM harness on macOS, Linux, and Windows. The Nix flake (see
  [Build & Run](#build--run)) pins the same toolchain locally as CI uses, so
  `just check` reproduces the pipeline byte-for-byte.

## Security

Faux Boy treats every byte that crosses into the emulator from outside as
adversarial input. ROMs come from the internet, save files come from disk,
and any of them can be malformed, attacker-crafted, or just plain corrupted.
The core has to handle that without crashing, looping, or invoking
undefined behavior.

This is **input-robustness security, not production hardening.** No
sandboxing, no code signing, no TLS — just "don't crash on a hostile ROM."

### Untrusted Surfaces

- **Cartridge ROM bytes.** Header lies about size, MBC writes carry
  attacker-controlled bank indices.
- **Battery RAM (`.sav` files).** On-disk file. Wrong size, swapped between
  carts, deliberately corrupted.
- **Save-state / snapshot files.** Same disk threat. Version mismatch,
  malformed structure, wrong cart fingerprint.
- **RTC state (MBC3).** Battery-backed clock data inside the cart RAM blob.
- **Boot ROM image.** User-provided (we never ship Nintendo's). Wrong size,
  corrupted, or swapped between DMG and CGB.
- **`.sym` symbol files.** A malformed line shouldn't crash the disassembler.
- **Tracelog / state-dump output paths.** User-supplied destinations get
  path-traversal validation; `--trace-out=../etc/passwd` is rejected.

### Engineering Rules

- **Checked arithmetic at boundaries.** `std.math.cast` for narrowing,
  `std.math.add` / `sub` / `mul` for size math. `@intCast` and `@truncate`
  are unchecked in `ReleaseFast` and `ReleaseSmall` — avoid on
  input-derived values.
- **Bounds-validate slice access on hot paths.** The most likely OOB
  sources are cart-header size fields the ROM lies about and MBC bank
  indices the ROM controls via memory writes.
- **Explicit endian reads.** `std.mem.readInt(u16, bytes, .little)` —
  never cast-and-pray on byte alignment.
- **Fail closed on save states.** Version mismatch, malformed structure,
  or cross-cart load → clean error, not silent state corruption.

### Fuzz as a Security Regression

The fuzz harness (see [Engineering](#engineering)) doubles as the standing
test that the rules above hold:

- Random-byte cart fuzz: garbage bytes, run for N cycles, assert no panic /
  UB / unbounded memory growth / infinite loop.
- Random-input fuzz on real ROMs: catches state-machine deadends in the
  CPU/PPU under malformed cart RAM writes.
- Save-state fuzz: random bytes loaded as a state, assert clean rejection.

### Out of Scope

- Process sandboxing (seccomp / App Sandbox / etc.)
- Code-signing release binaries
- Supply-chain auditing beyond Nix flake hash-pinning
- Formal CVE disclosure process

## Build & Run

Faux Boy targets Zig **0.16.0**. The repo ships a Nix flake that pins the
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
