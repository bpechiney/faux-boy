---
name: emulator-diagnosis
description: Disciplined diagnosis loop for cycle-accurate Game Boy, NES, and SNES emulator bugs and performance regressions. Build feedback loop → reproduce → hypothesise → instrument → fix → regression-test, with a symptom→cause dictionary as Phase 3 hypothesis seed. Use when the user says "debug this" / "diagnose this" for emulator code, reports a bug under cpu/, ppu/, apu/, cart/, mappers/, or bus/, describes a performance regression in the dispatch loop / pixel pipeline / sample generator, or names a symptom — "hangs", "audio sounds wrong", "save state restores wrong", "PAL build broken", "title screen freezes", "graphics garbled".
---

# Emulator Diagnosis

A discipline for hard cycle-accurate emulator bugs. Skip phases only when explicitly justified.

When exploring the codebase, use the project's `CONTEXT.md` and `docs/adr/` for domain context (hardware codenames, fidelity scope, mapper revision policy, region threading). For citation taxonomy (`QUIRK` / `HW` / `HACK` / `TODO` / `REF` / `TEST`) and the typed `Hacks` namespace conventions used in Phases 4–6, see `/emudev`.

## Phase 1 — Build a feedback loop

**This is the skill.** Everything else is mechanical. If you have a fast, deterministic, agent-runnable pass/fail signal for the bug, you will find the cause — bisection, hypothesis-testing, and instrumentation all just consume that signal. If you don't have one, no amount of staring at code will save you.

Spend disproportionate effort here. **Be aggressive. Be creative. Refuse to give up.**

### Ways to construct one — try them in roughly this order

**Setup cost varies — verify the harness before suggesting a technique.** Zero-setup techniques (save-state round-trip equivalence, throwaway harness, bisection, conditional breakpoints) work in any repo. Repo-infrastructure techniques (test-ROM result-address assertion, golden-frame hash diff, controller-input replay) require the project to have wired the test-ROM submodule, snapshot directory, or input-log format. External-tooling techniques (CPU-trace differential, reference-emulator differential) require a reference emulator binary *and* a known trace-export invocation already wired into the project — Mesen 2 / SameBoy / ares don't drive themselves. Check `docs/agents/emudev.md` and the repo layout for what's actually available. If the harness for a technique doesn't exist, drop to a lower-cost technique or build the harness as Phase 1 work — don't pretend a technique is usable when it isn't.

1. **Test-ROM result-address assertion.** Run a known test ROM (blargg, mooneye-test-suite, nestest, blargg APU/PPU suites, fullsnes test pack) and read the documented result byte (e.g. `$6000`/`$6004` for blargg, `$F000` for nestest, OAM/VRAM dumps for PPU tests). Assert pass/fail. The strongest emulator feedback loop when one exists.
2. **CPU-trace differential.** Run an instruction trace against a reference log (`nestest.log`, BGB / SameBoy trace dump, Mesen 2 trace log for SNES). Diff per cycle. The first diverging cycle is the bug.
3. **Golden-frame hash diff.** Boot deterministically, run N frames, hash the framebuffer, compare to a stored reference snapshot. Fail loudly on divergence.
4. **Reference-emulator differential.** Same ROM, same input, same N cycles through a citation-grade reference (Mesen for NES, BGB / SameBoy for Game Boy, Mesen 2 / ares for SNES). Diff observable state — CPU registers, PPU/APU register file, framebuffer hash.
5. **Save-state round-trip equivalence.** `save → load → run K cycles` must produce the same hash as `run K cycles` from the unsaved baseline. A diff exposes implicit state missing from the serializer.
6. **Controller-input replay.** Record a deterministic input log against a real ROM scenario (boot to title, press Start, walk left until tile X triggers); replay it through the emulator with frame-perfect timing.
7. **Bisection harness.** If the bug appeared between two known states (commit, dataset, mapper revision), automate "boot at state X, check, repeat" so you can `git bisect run` it.

See `/emudev`'s `testing.md` for the canonical patterns — test-ROM root layout, golden-trace harness, determinism tests.

Build the right feedback loop, and the bug is 90% fixed.

### Iterate on the loop itself

Treat the loop as a product. Once you have _a_ loop, ask:

- Can I make it faster? (Cache setup, skip unrelated init, narrow the test ROM, run fewer frames.)
- Can I make the signal sharper? (Assert on the specific symptom — exact result byte, exact framebuffer region, exact cycle of divergence — not "didn't crash".)
- Can I make it more deterministic? (Pin time, seed RNG, force uninitialised RAM to a fixed pattern, freeze any host-clock-derived inputs.)

A 30-second flaky loop is barely better than no loop. A 2-second deterministic loop is a debugging superpower.

### Non-deterministic bugs

The goal is not a clean repro but a **higher reproduction rate**. Loop the trigger 100×, parallelise, narrow timing windows, inject sleeps at suspect boundaries. A 50%-flake bug is debuggable; 1% is not — keep raising the rate until it's debuggable.

**In an emulator, non-determinism is itself a bug.** Host RNG, uninitialised RAM pulled from the host allocator, hash-map iteration order, time-based seed, host-endian `@bitCast` — any of these leaking into observable state breaks every other technique in this skill. If your repro flakes, *first* hunt the non-determinism, *then* hunt the original bug.

### When you genuinely cannot build a loop

Stop and say so explicitly. List what you tried. Ask the user for: (a) a controller-input log + ROM identifier that reproduces it, (b) a captured save-state from just-before-failure, (c) a CPU/PPU trace from a reference emulator at the failure point, or (d) permission to add temporary instrumentation to a build the user can run on their target ROM. Do **not** proceed to hypothesise without a loop.

Do not proceed to Phase 2 until you have a loop you believe in.

## Phase 2 — Reproduce

Run the loop. Watch the bug appear.

Confirm:

- [ ] The loop produces the failure mode the **user** described — not a different failure that happens to be nearby. Wrong bug = wrong fix.
- [ ] The failure is reproducible across multiple runs. **In an emulator, "reproducible" means bit-identical** — same instruction trace, same framebuffer hash, same audio sample stream, same final state. Anything less means non-determinism is leaking in; treat that as a Phase 1 problem before continuing.
- [ ] You have captured the exact symptom (test-ROM result code, frame hash, audio sample dump, freeze cycle number, diverging instruction PC) so later phases can verify the fix actually addresses it.

Do not proceed until you reproduce the bug.

## Phase 3 — Hypothesise

Generate **3–5 ranked hypotheses** before testing any of them. Single-hypothesis generation anchors on the first plausible idea.

Each hypothesis must be **falsifiable**: state the prediction it makes.

> Format: "If <X> is the cause, then <changing Y> will make the bug disappear / <changing Z> will make it worse."

If you cannot state the prediction, the hypothesis is a vibe — discard or sharpen it.

**Seed hypotheses from `failure_diagnosis.md`.** The dictionary is a symptom→causes lookup, ranked by likelihood, platform-tagged (`[NES]` / `[GB]` / `[SNES]` / `All`). If your symptom matches an entry (or is close), the listed causes are your initial ranked hypotheses — walk top-down. Add platform-specific candidates the dictionary doesn't yet cover; if the cause turns out to be one of those, add it back to the dictionary in Phase 6.

**Show the ranked list to the user before testing.** They often have domain knowledge that re-ranks instantly ("we just landed an MMC3 IRQ refactor — try #3 first"), or know hypotheses they've already ruled out. Cheap checkpoint, big time saver. Don't block on it — proceed with your ranking if the user is AFK.

## Phase 4 — Instrument

Each probe must map to a specific prediction from Phase 3. **Change one variable at a time.**

Tool preference for emulator work:

1. **Cycle-stamped trace logs.** Tag every probe `[DEBUG-a4f2][cycle=12345]` so you can correlate across CPU, PPU, APU, mapper. The cycle is the contract; without it, multi-component bugs are indistinguishable.
2. **MMIO write/read log.** Every write to PPU / APU / mapper / coprocessor registers with cycle stamp and PC. Most cycle-accuracy bugs are visible here — wrong order, wrong value, wrong cycle.
3. **Component isolation.** Replace the suspect component with a known-good reference implementation (or a trivial pass-through). If the bug disappears, the suspect is wrong; if it persists, eliminate that component and look elsewhere.
4. **Test-ROM binary search.** If a long test ROM fails late, bisect: run a smaller ROM that exercises only the suspect subsystem. Most major test-ROM suites are split this way already (`cpu_instr/01-implied`, `cpu_instr/02-immediate`, …) — start at the boundary closest to the suspect.
5. **Conditional debugger breakpoints** if the env supports it. One breakpoint at `cycle == 423891 && PC == 0xC1F0` beats ten log lines.

Never "log everything and grep". Each probe answers one prediction.

**Tag every debug log** with a unique prefix, e.g. `[DEBUG-a4f2]`. Cleanup at the end becomes a single grep. Untagged logs survive; tagged logs die.

**Perf branch.** For performance regressions, logs are usually wrong. Instead: establish a baseline measurement (Tracy, `perf record`, frame-time histogram, dispatch-loop microbench), then bisect. Common emudev hot spots:

- CPU dispatch loop — LLVM inlining across switch arms, mispredicted branches, unnecessary indirection.
- PPU pixel pipeline — per-pixel allocations, branch-heavy sprite priority resolution, cache-cold tile fetches.
- APU sample generator — resampler buffer churn, allocations on the audio thread, denormals.
- Bus / memory map — vtable indirection on hot path, mapper-lookup branches, mirror-resolution arithmetic.

Measure first, fix second.

## Phase 5 — Fix + regression test

Write the regression test **before the fix** — but only if there is a **correct seam** for it.

A correct seam is one where the test exercises the **real bug pattern** as it occurs at the call site. In emudev, the correct seam is usually one of:

- **Test-ROM assertion** — a passing test ROM whose result address now reads "ok". Strongest seam when one exists.
- **Golden-frame hash** — a stored framebuffer hash at frame N after deterministic boot.
- **Save-state round-trip property** — `save → load → run K → hash` equals `run K → hash`.
- **Cycle-trace equality** — diff against a reference log up to cycle N.
- **Unit test** — only when the bug is genuinely localised (one mapper register's bit semantics, one addressing-mode calculation). Most emudev bugs cross component boundaries; a unit test there gives false confidence.

If the only available seam is too shallow (a unit test that can't replicate the multi-component chain that triggered the bug, or a test-ROM result that's coincidentally also "pass" when the bug is present), a regression test there gives false confidence.

**If no correct seam exists, that itself is the finding.** Note it. The codebase architecture is preventing the bug from being locked down. Flag this for Phase 6.

If a correct seam exists:

1. Turn the minimised repro into a failing test at that seam.
2. Watch it fail.
3. Apply the fix.
4. Watch it pass.
5. Re-run the Phase 1 feedback loop against the original (un-minimised) scenario.

For test-driven cycles where the fix unfolds across multiple commits (e.g. a save-state schema change), drive the red-green-refactor loop via `/tdd`.

## Phase 6 — Cleanup + post-mortem

Required before declaring done:

- [ ] Original repro no longer reproduces (re-run the Phase 1 loop)
- [ ] Regression test passes (or absence of seam is documented)
- [ ] All `[DEBUG-...]` instrumentation removed (`grep` the prefix)
- [ ] Throwaway prototypes deleted (or moved to a clearly-marked debug location)
- [ ] **The corrected hypothesis becomes a citation comment** at the fix site, per `/emudev`'s six-tag taxonomy: `// QUIRK` for universal hardware quirks the cycle-accuracy tier dictates you reproduce, `// HW` for hardware behaviour the code reflects, `// HACK[Game] (#issue)` for temporary band-aids (bracketed game name, linked issue, and stated hardware uncertainty — all three required), `// REF` for spec-derived behaviour. Cite nesdev wiki, pandocs, fullsnes, or a named test ROM. Reputation-based "this game is picky" attributions are folklore — don't ship them.
- [ ] If the cause was previously undocumented and falsifiable, **add a new entry to `failure_diagnosis.md`** for the next debugger.

**Then ask: what would have prevented this bug?**

- If the answer is architectural (no good test seam, tangled callers, hidden coupling between components, save-state schema couldn't represent the implicit state), hand off to `/improve-codebase-architecture` with the specifics. Make the recommendation **after** the fix is in, not before — you have more information now than when you started.
- If the answer is "we never made this load-bearing decision explicitly" (cycle-accuracy tier, save-state schema versioning, inter-CPU coordination scheme, fidelity scope), walk it via `/grill-with-docs` to produce an ADR.
