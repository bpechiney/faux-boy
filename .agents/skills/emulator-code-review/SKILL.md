---
name: emulator-code-review
description: Reviews Zig code for cycle-accurate Game Boy, NES, and SNES emulators with emphasis on hardware-quirk fidelity, cycle-level timing, and test-ROM compliance. Layers on top of generic Zig review — does not duplicate it. Use when reviewing files under cpu/, ppu/, apu/, cart/, mappers/, or bus/; when the user mentions opcodes, addressing modes, T-states, M-cycles, scanlines, vblank, hblank, mappers, MBCs, IRQs, NMIs, OAM, DMC, or frame counters; or when the user explicitly asks for review of NES/Game Boy/SNES emulator code.
---

# Emulator Code Review

Review posture for cycle-accurate retro console emulators in Zig 0.16+. Out of scope: leaks, format strings, allocator ergonomics, post-Writergate idioms — see anti-feature 5.

## Methodology — eight rules

1. **Defer to test ROMs over code aesthetics.** Ugly code that matches `nestest.log` is correct; elegant code that fails Blargg is wrong. In this domain, treat elegance as a smell until it's been run against the suite.
2. **Ask "what cycle does this happen on?" before "is this code clean?"** Every memory access, flag update, and interrupt poll has a cycle number that's part of the contract with the rest of the system. Lose the cycle, lose the contract.
3. **Read for missing behaviour, not just present bugs.** The hardest emulator bugs are omissions. Walk the per-component checklist explicitly; don't only react to what's on screen.
4. **Determinism is a separate axis from cycle accuracy.** Flag `std.AutoHashMap` iteration leaking into observable state, time-based seeding, threading without ordered scheduling, host-endian-dependent `@bitCast`, uninitialised RAM patterns that vary per run. Required for TAS, netplay, save-state rewind, and golden-frame regression tests.
5. **State-modelling completeness.** For each component, ask: "what mutable state survives a save/load that isn't an obvious named field?" Implicit state hides in control flow — in-progress instruction cycle counter, edge-triggered interrupt latches, OAM/address latch toggles, partial 16-bit register writes, in-flight DMA progress, APU frame-counter sub-state, open-bus decay value. "Nothing implicit" is almost always wrong.
6. **Comments-as-citations are a positive signal — opposite to default Zig style.** Look for citations of nesdev wiki, pandocs, fullsnes; named test ROMs; named games that depend on the quirk; matched hardware revision. The bug to flag is *uncited* quirky code, not the citations.
7. **Region/revision is a review axis with named consequences.** Hardcoding region silently breaks specific behaviour: a hardcoded master-clock divider produces the wrong CPU rate on the other region (NES NTSC ÷12 vs PAL ÷16), a hardcoded OAM-bug path runs DMG behaviour on CGB (the bug is silicon-revision-gated, not universal), a hardcoded open-bus model misses the 1-CHIP SNES delta, and a hardcoded DMC rate table desyncs PAL audio (NTSC vs PAL DMC period tables differ). Ask: is region threaded as a parameter through every component that consumes it (CPU divider, PPU scanline count, APU rate table, mapper-IRQ tick source, OAM-bug gate), or is it hardcoded somewhere it'll bite later? 2A03 vs 2A07, DMG/MGB/SGB/CGB/AGB, original vs 1-CHIP SNES — each split has at least one cycle-observable consequence.
8. **Falsifiability check at finding-formation.** For each finding, briefly state what evidence would disprove it. If disproof is impossible or fuzzy, the finding is speculation — downgrade evidence by one level. The reviewer's claim is falsifiable, or it's not a finding. This is the downstream protection paired with the fresh-context invocation discipline (see *Invocation context* below): fresh context prevents anchored priors; falsifiability prevents anchored conclusions.

## Anti-features — what this skill will NOT do

- **No architectural redesign suggestions.** This is a code-review skill, not a design-review skill. Take the existing architecture as given — per-cycle state machine vs micro-op table vs catch-up scheduler, vtable vs tagged union for mappers, threading model, module boundaries. Review the code *within* that architecture. Architectural concerns are a separate design-review pass; checklists like `cycle_accuracy.md` describe **what bugs to look for in code written in each style**, not how to choose between styles.
- **No "rewrite this faster" suggestions.** Performance is a separate review pass.
- **No code-style enforcement that fights hot-path patterns.** A 256-case labelled `switch` with `continue :dispatch .next` is the *correct* shape for an interpreter loop in Zig 0.16+; do not suggest extracting opcode handlers into functions, because LLVM cannot inline across the indirection.
- **No manufacturer datasheets as authoritative sources.** Cite nesdev wiki, pandocs, fullsnes, and test ROMs — these reflect observed silicon, not marketing intent.
- **No generic Zig review.** Out of scope: leaks, allocator hygiene, format strings, error-set surface area, post-Writergate idioms. Flag findings in those areas as out-of-scope rather than reviewing them. The exception is Zig 0.16-specific patterns that affect cycle-accurate code paths: labelled-switch dispatch with `continue :dispatch .next`, and exhaustiveness checks on per-cycle state-machine enums. Those are inlined where they bite — see `checklists/cycle_accuracy.md` §7.

## Routing — load only what the review needs

Always load the relevant **component checklist** for the file under review. Add `checklists/cycle_accuracy.md` for any timing-sensitive review, and `references/test_roms.md` when the user references a specific ROM result. Per-system citation lists live at `references/{nes,gameboy,snes}.md` — load these when a finding needs an external citation pointer; checklists already cite inline.

The rows below dispatch checklists, not reviewers. A PR touching multiple modules needs one reviewer holding all the relevant checklists, not separate reviewers per module — most insidious emulator bugs cross module boundaries (DMC sample-fetch CPU stealing, MMC3 IRQ shifting sprite-0 hit timing, region threading between CPU divider and OAM-bug gate). Split per module only if the diff is too large to hold; otherwise one reviewer with the full diff is the default.

| Reviewing... | Load |
|---|---|
| NES CPU (6502 / 2A03) | `checklists/cpu_review.md` + `checklists/cycle_accuracy.md` |
| NES PPU | `checklists/ppu_review.md` |
| NES APU | `checklists/apu_review.md` |
| NES mapper / cart | `checklists/mapper_review.md` |
| Game Boy CPU (LR35902) | `checklists/cpu_review.md` + `checklists/cycle_accuracy.md` |
| Game Boy PPU / OAM DMA | `checklists/ppu_review.md` + `checklists/bus_review.md` |
| Game Boy MBC | `checklists/mapper_review.md` |
| SNES 65C816 / SPC700 | `checklists/cpu_review.md` + `checklists/cycle_accuracy.md` |
| SNES PPU / HDMA | `checklists/ppu_review.md` + `checklists/bus_review.md` |
| Bus / memory map | `checklists/bus_review.md` |
| Save state / serializer | `checklists/save_state_review.md` (and re-skim each component checklist for implicit state) |
| Cartridge / ROM header parser | `checklists/mapper_review.md` §7 |
| Test-ROM harness | `checklists/test_rom_harness.md` + `checklists/save_state_review.md` §5 |
| Scheduler / clock-sharing | `checklists/cycle_accuracy.md` §4 + `checklists/bus_review.md` |
| Hot-path performance, architectural redesign | **out of scope** — see anti-features |
| Frontend glue, CI, save-directory / config plumbing | **out of scope** |
| `build.zig` / `build.zig.zon` (artifact split, feature flags, test wiring) | **out of scope** |

## Invocation context

This skill is designed to run with **minimal context**: the code under review, the architecture as a one-line stated fact (e.g. "per-cycle state machine, mappers as vtable"), and the review scope (files or PR). Do **not** pass the implementation conversation that produced the code — it anchors the reviewer on the author's framing and degrades the suspicion posture rules 1 ("suspect elegance") and 3 ("read for missing behaviour") require. Do **not** pass the author's self-assessment of risk ("I'm worried about IRQ timing") — route focused review via the relevant checklist instead. Prefer sub-agent invocation with a fresh context window over in-context invocation in the session that wrote the code.

Two consequences of fresh-context invocation:

- **Expect a higher false-positive rate on intentional-but-undocumented omissions.** A fresh reviewer will flag a missing undocumented opcode that the author deliberately skipped because the target ROM set doesn't use it. The fix is citation discipline (rule 6) — a code comment resolves the finding as "intentional, cited." Do not compensate by feeding more context to the reviewer.
- **Confidence ratings (below) are meaningful only under fresh-context invocation.** A reviewer whose priors were shaped by the implementation conversation isn't producing an independent estimate. In-context invocation silently invalidates the confidence field.

## Output shape

Group findings by methodology rule (cycle / missing-behaviour / determinism / state / citation / region). Within each group, sort by evidence: ✅ first, then ⚠️, then ❌.

For each finding:

1. **What's wrong or missing** (file:line if possible).
2. **Why it matters** — the hardware behaviour it violates with citation (nesdev / pandocs / fullsnes), and either a named game or scenario that depends on it OR "no specific game known — structural finding" when the violation is structural (determinism, save-state field omission, etc.).
3. **Evidence** — one of:
    - **✅ Verified** — names the test ROM, result address, and pass/fail interpretation that confirms the bug.
    - **⚠️ Documented** — cites the canonical source describing the violated behaviour (nesdev / pandocs / fullsnes), with a minimal reproducer or unit-test sketch. No test ROM directly catches this yet.
    - **❌ Speculation** — no test ROM, no canonical source directly speaks to this; the claim is pattern-match or intuition. Findings at this level must not include a fix — flag as research-needed.
4. **Intent ambiguity?** — yes / no. If yes, the finding could be intentional-but-undocumented (silicon quirk preserved deliberately, optimization that drops a flag, etc.); the fix is likely citation discipline (rule 6 — add a citation comment so the intent is explicit) rather than a code change.

Apply rule 8 (falsifiability) before locking in the evidence level: if you can't state what would disprove the finding, downgrade to the next level down.
