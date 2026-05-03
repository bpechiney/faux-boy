---
name: emudev
description: Coding standards for cycle-accurate retro console emulators (Game Boy, NES, SNES) in Zig 0.16. Use when working under cpu/, ppu/, apu/, cart/, mappers/, bus/, or when the user mentions opcodes, addressing modes, T-states, M-cycles, scanlines, vblank/hblank, mappers, MBCs, IRQs, NMIs, OAM, DMC, frame counters, save-state schemas, or test ROMs. Loads standing rules and the per-repo configuration in docs/agents/emudev.md.
---

# Emudev

Coding standards for cycle-accurate retro console emulators (Game Boy, NES, SNES) in Zig 0.16.

## At session start

1. Read `docs/agents/emudev.md` to find the active `system` (`gameboy`, `nes`, `snes`) and other config (cycle-accuracy tier, `Hacks` location, test-ROM root).
2. If the file is missing, run the lazy-creation interview from [setup.md](./setup.md). It is idempotent — re-running on an existing config edits in place rather than overwriting.
3. Verify Zig version. Read `build.zig.zon`'s `minimum_zig_version`. If it isn't `0.16.x`, surface a one-line warning at session start and proceed using 0.16 conventions.

System-specific domain content (hardware codenames, test-ROM wiring, fidelity-scope choices) lives in the consuming emulator repo's `CONTEXT.md` / `docs/` / `docs/adr/` — not in this skill. The skill is about *how* to write emulator code; the *what* of any specific system is the consuming repo's territory.

## Standing rules

These are non-negotiable across every emudev repo. They are the part of this skill that should not be argued with mid-implementation.

### 1. Six-tag citation taxonomy

Every function or block whose existence is hardware-derived carries at least one tag. The six tags: `REF`, `QUIRK`, `HW`, `TEST`, `HACK`, `TODO`. Full taxonomy, examples, and grep recipes live in [comments.md](./comments.md).

The proximity rule is **per-function-or-block**, not per-N-lines. Mechanical regions (long opcode tables) need one tag at the top, not one per ten lines.

### 2. `HACK` requires three things

A `// HACK[Game] (#issue): ...` is only valid if it carries:

1. Bracketed game or test name — what real-world thing forced this.
2. Linked issue — tracking the removal target.
3. Stated hardware uncertainty — what we don't yet know that justifies the imperfection.

If any of the three is missing, **it isn't a HACK — it's bad code**. Delete it instead of tagging it.

### 3. `TODO` requires linked issue

Every `// TODO(#N): ...` carries a linked issue. No floating TODOs.

### 4. No allocations in the hot path

The CPU dispatch loop, PPU pixel pipeline, and APU sample generator do not allocate. Pre-allocate at boot; pass allocators only into setup paths and frontend boundaries.

### 5. Typed `Hacks` namespace from day one

Every emudev repo has a typed `Hacks` namespace. Location declared in `docs/agents/emudev.md`. Every active hack is named there with a removal target. Inline `HACK[Game]` tags at use sites are the breadcrumbs; the namespace is the catalog. The point is auditability — untyped, scattered hacks become invisible debt; a typed catalog stays prosecutable.

## Seven decisions worth grilling

Before implementing a new emulator (or a major subsystem), invoke `/grill-with-docs` to walk these candidates. Each typically passes the three-test (hard-to-reverse + surprising + real-trade-off) — `grill-with-docs` decides whether each warrants an ADR for *this* repo. Decision #7 only applies when the design includes more than one programmable CPU; designs with a single CPU can skip it.

1. **Dispatch strategy** — labeled `switch` with `continue :dispatch op` (the Zig 0.16 idiom; +13% on Zig's own tokenizer; observed in 0/12 surveyed Zig emulators) vs function-pointer table vs giant `switch`. See [dispatch.md](./dispatch.md).

2. **Mapper polymorphism** — tagged `union(enum)` with `inline else` (closed historical sets — NES has ~250 mappers but it's a closed set) vs vtable (open frontends) vs hybrid. See [polymorphism.md](./polymorphism.md).

3. **Cycle accuracy tier** — instruction-stepped vs M-cycle vs T-state. Hard to upgrade later (changing tier rewrites the CPU loop and ripples to PPU/APU).

4. **Save-state schema versioning + migration** — explicit version + migration ladder vs schema-as-code (compile-time snapshot of the state struct) vs deferred. Bumping a version without a migration breaks every user save. See [testing.md](./testing.md).

5. **CPU↔Bus boundary** — `comptime Bus: type` (monomorphized per concrete bus type) vs vtable (runtime swap). Touches every instruction call — easier to commit early than to refactor later.

6. **Fidelity scope + gating mechanism** — which hardware revisions, regions, peripherals/accessories, boot ROMs, in-scope cartridge mappers, and analog characteristics this emulator faithfully reproduces, and how scope-dependent paths are gated (compile-time `comptime` vs runtime field). Concrete per-system candidates live in the consuming repo's `docs/adr/` (or `CONTEXT.md` for stable canon like hardware codenames). Entangled with #4 (save-state must encode the active revision) and #2 (revision gating may reuse the tagged-union pattern from mapper polymorphism).

7. **Inter-CPU coordination** *(multi-CPU systems only)* — when the system has more than one programmable CPU running on independent clocks (SNES main 65816 + SPC700 audio coprocessor; cartridge coprocessors like SuperFX or SA-1; SGB-mode Game Boy under SNES host), pick a scheme:
    - **Catch-up scheduler** — each CPU runs ahead independently; on inter-CPU communication, the lagging CPU is stepped forward to sync. Fastest to write; usually fastest to run; can drift on tight inter-CPU timing.
    - **Cycle-locked stepping** — every minimum clock tick advances all CPUs together. Most accurate; slowest; the high-fidelity-emulator school.
    - **Coroutine-based** — each CPU is a coroutine that yields at sync points. Splits the difference; requires explicit yield-machinery and careful save-state handling (resumable coroutines complicate serialization).

    Hard to reverse — the choice shapes the entire emulation control flow, the save-state schema (decision #4), and the per-CPU bus boundary (decision #5). Skip when the design has only one programmable CPU; load-bearing when there are two or more (typical for SNES; for Game Boy and NES, single-CPU is the usual baseline but expansion-coprocessor scope choices in decision #6 can introduce a second CPU and bring this decision back into play).

Note: "fidelity scope" (the ADR-level scope choice) is distinct from `QUIRK` (the inline tag for universal hardware quirks the cycle-accuracy tier dictates you reproduce regardless). See [comments.md](./comments.md) for the naming hygiene.

## Composition with sibling skills

Emudev defers loops and processes to sibling skills. It contributes domain content; it does not parallel their orchestration.

- **`/tdd`** drives the red-green-refactor loop. Emudev provides *what to test against* (test ROMs, golden traces, determinism, save-state round-trip — see [testing.md](./testing.md)) and *how to write the implementation* (citations, dispatch, packed structs).
- **`/grill-with-docs`** walks the seven load-bearing decisions above. Emudev does not write ADRs directly.
- **`/to-issues`** slices implementation work into vertical tracer-bullet issues. Emudev does not parallel its slicing logic.
- **`/emulator-diagnosis`** runs the hardware-quirk debugging loop. Emulator dev is bug-hunt-heavy; `/emulator-diagnosis` is the daily driver.
- **`/improve-codebase-architecture`** reads ADRs produced via `/grill-with-docs`. No direct integration.
- **`/triage`** suggested labels for emulator work: `cycle-accuracy`, `mapper-compat`, `test-rom-failing`, `hack-debt`, `cite-needed`. Configure in `docs/agents/triage-labels.md` per `/setup-matt-pocock-skills`.

`/tdd`, `/emulator-diagnosis`, and `/improve-codebase-architecture` carry one-line cross-references to `/emudev` so the agent picks up domain context when triggered by the sibling skill.

## Zig version

This skill assumes **Zig 0.16** (released 2026-04-13). All code samples target 0.16.

A Zig major-release bump is not a version-flag flip — it's a content re-audit of every code sample and recommended pattern. Each recent Zig release has reshaped at least one of:

- The build-system surface (`b.addOptions`, `b.addModule`, `addImport` shapes)
- Labeled-switch / `continue :state` syntax (the dispatch pattern recommended here is itself recent)
- `inline else` semantics on tagged unions
- `@bitCast` / `@ptrCast` / `@enumFromInt` builtins and their alignment rules
- `packed struct(T)` declaration shape
- Stdlib reorganization

Plan a Zig migration as a focused PR against this skill: re-validate every code sample under the new compiler, update idioms where the language has moved, then bump the version pin. Don't let consuming repos drift ahead of the skill — drift between the two is what breaks "works in my emulator repo" assumptions.

If `build.zig.zon`'s `minimum_zig_version` is not `0.16.x`, surface a one-line warning at session start. Proceed using 0.16 conventions; do not attempt to adapt code samples to other versions on the fly.

## File index

| File | Use when |
|---|---|
| [comments.md](./comments.md) | Adding inline citations, writing or reviewing comments |
| [dispatch.md](./dispatch.md) | Implementing CPU opcode dispatch |
| [polymorphism.md](./polymorphism.md) | Implementing mapper variants or any closed-set polymorphism |
| [packed-structs.md](./packed-structs.md) | Modeling hardware registers and sprite tables |
| [testing.md](./testing.md) | Wiring test ROMs, determinism tests, save-state round-trip |
| [build.md](./build.md) | Editing `build.zig` (artifact split, feature flags, test wiring) |
| [setup.md](./setup.md) | First emudev invocation in a fresh repo |
