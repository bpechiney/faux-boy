# Test-ROM Harness Review Checklist

Covers PRs that touch the **test-ROM harness**: the wiring that runs a test ROM under the emulator core, observes its output, and asserts pass/fail. This is review of the *harness*, not of the emulator code under test — a harness bug silently masks emulator bugs (or worse, produces flaky CI that erodes the suite's authority).

The harness is its own surface: it has its own deterministic-execution requirements, its own oracle choices, and its own assertion-strictness ladder. Review it like any other test infrastructure — but with the extra discipline that the *thing being tested* is hardware-cycle-accurate, so the harness itself must not introduce non-determinism on top.

Out of scope: per-ROM identity tuples (what each ROM tests, where its result byte lands). That is standing-fact and lives in the consuming repo's harness or in the ROM family's upstream docs. This checklist reviews the harness *shape*, not the ROM catalog.

Cross-reference [`save_state_review.md`](save_state_review.md) §5 — round-trip determinism is the analogue test for save-state, and the same hash-discipline applies here.

---

## 1. Result-byte interpretation per ROM family (methodology rule 1)

Each ROM family has its own convention for "the test passed." A harness that hardcodes one family's protocol silently mis-reports the others. The bug-shape is a green test that never actually checked the ROM's reported result — the ROM ran for N cycles, the harness assumed pass, no oracle was consulted.

**Blargg-style (`$6000` / `$6001` / `$6004+` text protocol) read from the right address with the right "in-progress vs final" state machine?**
Blargg roms write `0x80` to `$6000` while the test runs and overwrite with the final result byte (`0x00` = pass) on completion. The harness must wait for the not-`0x80` transition, *then* sample the result byte and the ASCII message starting at `$6004`. A harness that samples `$6000` once and asserts `== 0` will pass any ROM that's still mid-test (because the value is still `0x80`, not `0x00` — wait, `0x80 != 0` so this case fails-safe; the failure-shape is the inverse: sampling once-and-done on a ROM that briefly writes `0x00` during init).

**Mooneye-style (registers-as-magic-values protocol) reads the right register set?**
Mooneye writes a known sequence (Fibonacci `3, 5, 8, 13, 21, 34` in `B/C/D/E/H/L`) and halts via `LD B, B` (fail) or `LD D, D` (pass) — interpretation requires harness-side instruction-stream awareness, not just memory polling. A harness reading only memory addresses misses mooneye's contract entirely.

**dmg-acid2 / cgb-acid2 / image-comparison roms produce a framebuffer, not a result byte — harness compares against a reference image?**
The oracle is a hash of the framebuffer at the documented "test complete" point (usually after the ROM halts via `STOP` or a known busy-loop). A harness that polls a result byte address on an image-comparison ROM is reading garbage. The harness must dispatch by ROM family, not run a single protocol.

**Default-on-unknown-family is "fail loudly," not "assume Blargg"?**
A harness that silently falls back to the most-common protocol when the ROM family is unrecognised converts misconfigurations into false passes. Unknown family → harness error, not test pass.

**Pattern-not-prescription:** the rule is "every result-byte read site dispatches on declared ROM family." Whether that's a tagged union, a config flag, or a per-suite harness invocation is implementation detail — the review test is whether a contributor can add a fourth ROM family without modifying the existing three protocol implementations.

---

## 2. Golden-frame-hash scope: when it's the right oracle (methodology rule 1)

Frame-hash oracles compare a hash of the framebuffer (and optionally audio buffer + system state) at a known cycle count against a stored reference hash. Right oracle in the right place; wrong oracle elsewhere.

**Frame-hash is the right oracle only when the ROM's *declared* output is visual or when the ROM doesn't expose a programmatic result?**
dmg-acid2, full_palette, ppu_vbl_nmi visualizations — these are made to be looked at. Frame-hash is correct.
Blargg cpu_test, mooneye acceptance — these declare a programmatic result. Frame-hash here is *redundant at best, wrong at worst*: it adds a non-deterministic surface (any UI overlay, frame-pacing tweak, palette LUT change) that has nothing to do with what the ROM is actually testing. The result byte is the source of truth; the frame is incidental.

**Frame-hash hides regressions when the bug doesn't perturb the captured pixels?**
Audio-only bugs, save-state bugs, bus-timing bugs that don't reach the pixel pipeline — none change the frame hash, so the oracle reports green while the regression ships. A frame-hash-only suite is not a substitute for per-component test-ROM coverage; it's a backstop for visual regressions specifically.

**Cycle of capture is documented and cycle-deterministic, not "after N seconds of wall time"?**
The frame-hash oracle is meaningful only if the captured frame is at a deterministic cycle. "Run for 5 seconds, hash the framebuffer" is wall-clock-flaky on any host. The recipe: run for exactly N master cycles, then hash. Cite [`cycle_accuracy.md`](cycle_accuracy.md) §3 — catch-up scheduler ordering — for why "run until cycle N" is itself a non-trivial discipline.

**Flaky-hash policy is explicit?**
A frame-hash test that fails intermittently is one of: (a) actual non-determinism in the emulator (rule-4 violation; the test is doing its job), (b) non-determinism in the *harness* (palette LUT swap, post-process overlay leaking into capture), (c) the reference hash was captured on a different build and the difference is intentional. Each has a different fix. The bug-shape is a flaky test silenced via retry-loop or `--allow-flaky`; that converts a real signal into noise. Review for: explicit triage path on hash mismatch, not silent retry.

**Reference hashes versioned alongside the harness?**
A reference hash stored without the build/commit it was captured from is undebugable when it later breaks. Each reference hash should be tied to (ROM file hash, harness commit, intentional-or-incidental-change marker). Cross-reference [`save_state_review.md`](save_state_review.md) §2 versioning — same discipline applies.

---

## 3. Acceptable assertion strictness (methodology rule 2)

Tests can assert at three levels of strictness. The harness should match the ROM family's claim — over-strictness produces flaky tests, under-strictness produces blind tests.

**Exact-cycle (T-state / M-cycle / dot exact) assertions are appropriate only when the ROM is cycle-asserting (mooneye `intr_*_timing`, ppu_vbl_nmi cycle-precise sub-tests)?**
Asserting cycle-exact correctness on a Blargg cpu test that only cares about the result byte is over-strict — a correct emulator that happens to retire instructions one cycle earlier than the reference timestamp will fail despite passing the test the ROM is actually running. Cycle-exact is the strictest level; reserve it for ROMs designed for it.

**Frame-accurate (output by cycle N matches reference) is the default for visual / audio ROMs?**
Visual ROMs (acid2, full_palette) and audio ROMs are inherently frame-accurate-by-construction: the output is sampled per-frame, so the assertion is per-frame. The harness must capture at frame boundaries deterministically (see §2).

**Output-only (final result byte / register / message) is correct for behavioural ROMs (Blargg cpu, mooneye acceptance)?**
The ROM's contract is "I will produce result X if my behaviour is correct"; the harness's job is to read X. Asserting anything else (cycle counts, intermediate framebuffer states, stack contents) is reading into the test's implementation, not its contract.

**Mixing strictness levels in the same harness has explicit per-test routing?**
A harness that runs all three families through the same assertion code path will either over-strict the relaxed ones (false fails) or under-strict the precise ones (false passes). Per-ROM-family routing is the correct shape; a single "run and hash" path is the bug-shape.

**Pattern-not-prescription:** the principle is "match the ROM's own claim." A ROM that asserts cycle-exact behaviour deserves a cycle-exact harness; a ROM that asserts a result byte deserves a result-byte harness. Don't strict-mode a relaxed test; don't relax a strict one.

---

## 4. Harness vs unit-test boundary

Test ROMs and unit tests answer different questions; a harness that conflates them either loses signal or duplicates work.

**Test-ROM harness asserts emergent behaviour from the integrated system; unit tests assert local invariants?**
A unit test on the CPU's `INC` opcode confirms the local arithmetic. The Blargg `cpu_test` confirms that *all* opcodes interact correctly with flags, memory, interrupts, and timing — many bugs only surface from interaction. A test-ROM harness reproducing what unit tests already cover is wasted CI time; a unit-test suite trying to cover what a test ROM covers is intractable.

**Unit tests own: per-opcode arithmetic, per-register decode, allocator-level memory operations, packed-struct round-trips. Test-ROM harness owns: integrated cycle accounting, multi-component IRQ/NMI sequencing, full-frame rendering, audio-pipeline output, mapper-IRQ-on-A12-in-context.**
Anything that requires the bus, CPU, PPU, APU, and mapper running together for hundreds of cycles is harness territory. Anything that fits in a `test "..." { ... }` block with deterministic input is unit-test territory.

**Unit tests run in the build (`zig build test`); harness runs separately, with explicit invocation and explicit time budget?**
Test ROMs are slow — minutes to hours for a full suite. Running them in `zig build test` couples the inner-dev-loop to the slow-suite execution. The bug-shape is a CI matrix that runs the whole ROM suite on every push and degrades to "tests take 40 minutes, devs stop running them locally." Harness invocation should be its own target / its own CI job.

**Failing test-ROM result includes enough context to route to a unit test, not just "ROM X failed"?**
The harness output is a triage signal, not a fix. "Blargg `cpu_dummy_writes_oam.nes` failed at cycle N with result byte 0x05" should let a developer locate the failure — typically by routing to the relevant component checklist (see §5 below). A bare "FAIL" with no context costs more time than the harness saves.

---

## 5. Failure-mode mapping back to component checklists

A failing test ROM is a *symptom*; the harness's role in diagnosis is to point at the right component checklist. The mapping below routes ROM-family failures to the checklist sections that catch them.

**Each ROM family / suite has a documented mapping from failure to relevant checklist?**
- Blargg `cpu_*` / `instr_*` / `branch_timing` failures → [`cpu_review.md`](cpu_review.md) §1 (cycle accounting), §2 (dummy reads/writes), §4 (interrupt polling).
- Blargg `ppu_*` / `sprite_*` / `oam_*` failures → [`ppu_review.md`](ppu_review.md) §2–§7 by sub-test name.
- Blargg `apu_test` / `dmc_*` / `length_counter` failures → [`apu_review.md`](apu_review.md) §1–§5 by sub-test.
- Blargg `mmc3_test/*` failures → [`mapper_review.md`](mapper_review.md) §1 (A12 filter), §2 (bank-switch timing).
- Mooneye `acceptance/timer/` failures → [`cpu_review.md`](cpu_review.md) §4 + GB-specific timer doc.
- Mooneye `acceptance/ppu/` failures → [`ppu_review.md`](ppu_review.md) §2 + GB pixel-FIFO references.
- Mooneye `acceptance/oam_dma/` failures → [`bus_review.md`](bus_review.md) §2 (DMA / CPU interaction).
- Mooneye `acceptance/mbc*/` failures → [`mapper_review.md`](mapper_review.md) §2, §4, §7.
- peter_lemon SNES tests (OpenBus / DMA / HDMA) failures → [`bus_review.md`](bus_review.md) §1 (open bus), §2 (DMA / HDMA channel ordering, WRAM-DMA conflict).
- Image-comparison failures (acid2 family) → [`ppu_review.md`](ppu_review.md) §3 (sprite-priority), §6 (OAM), §7 (background fetch).
- Round-trip / save-state harness failures → [`save_state_review.md`](save_state_review.md) §1, §5.

**Mapping is *in the harness output*, not in tribal knowledge?**
The bug-shape is a CI log that says "mooneye/acceptance/timer/tima_reload-GS failed" with no pointer to where to look. The harness should print the relevant checklist section (or at minimum a link / reference) so a developer who has never seen this failure before can begin diagnosis.

**Mapping is reviewed when a new test family is added?**
Adding a new ROM suite without updating the failure-mode mapping creates an unrouted failure: the test fails, the harness reports it, but no checklist is named. New ROM family → new mapping entry; reviewed before the suite lands.

**Pattern-not-prescription:** any sufficiently-deterministic mapping shape works (lookup table, string-matching on test name, per-suite handler). The review test is whether a triage routing exists at all — not whether it takes any specific shape.

---

## 6. Determinism intersection (methodology rule 4)

The harness itself must be deterministic, or the suite's authority erodes. Cross-reference each component's determinism section for the per-component cases; the harness-specific surface is below.

**Harness-side RNG seeded deterministically (or absent)?**
A harness that uses host-RNG to pick test order, sample timing, or anything else is the test of [`cpu_review.md`](cpu_review.md) §7 turned on the harness itself. Deterministic seed or no RNG.

**Test order is deterministic (no `std.AutoHashMap` iteration of test names)?**
Same rule as the per-component determinism heuristic: hash-map iteration order must not leak into observable behaviour, including the order tests run.

**Captured frame / hash / output is timestamped or cycle-counted, not wall-clocked?**
"Run for 5 seconds" → flaky. "Run for N master cycles" → deterministic. The harness's run-loop must count cycles, not poll wall time.

**Reference hashes regenerated only on intentional change, with the change flagged in the commit?**
A passing harness that quietly accepts a new reference hash on every run is not a test — it's a recorder. The bug-shape is `--update-snapshots` baked into CI.

---

## 7. Citation hygiene (methodology rule 6)

Test ROMs themselves are citations — name the ROM and the version. The harness inherits this discipline.

**Each harness assertion that depends on a specific ROM names the ROM and the result-byte protocol it uses?**
"This test expects `$6000 == 0` after `$6000` transitions away from `0x80`" cites the Blargg protocol. A harness assertion with a magic address and no comment is the bug-shape.

**Reference hashes carry provenance: ROM file hash + emulator commit + capture date?**
A reference hash without provenance is folkloric. When the hash later mismatches, no one can tell whether the emulator regressed, the ROM was rebuilt, or the harness changed.

**Failing test reports include the ROM family, ROM name, expected oracle, observed oracle, and pointer to the relevant checklist?**
This is citation discipline applied to test failures: "fails with no comment" is the bug-shape.

---

## 8. Test-ROM harness ↔ save-state harness symmetry

The save-state round-trip test (in [`save_state_review.md`](save_state_review.md) §5) is *itself* a test-ROM-harness consumer — it uses a deterministic test ROM as the deterministic-execution oracle for save/restore. The two harnesses should share infrastructure, not reimplement.

**Save-state round-trip uses the same cycle-counted run-loop as the per-suite harness?**
A save-state round-trip that wall-clocks "run for 5 seconds" while the per-suite harness counts cycles will produce two different determinism stories. Share the run-loop.

**Frame-hash / state-hash oracle in the save-state test is the same hash function and same capture point as the per-suite harness's frame-hash oracle?**
Different hash discipline in two harnesses → two reference-hash files to maintain → drift. Use one.

**The harness exposes "run to cycle N then return state" as a primitive, not buried inside per-test scaffolding?**
A harness whose run-loop is wired only into the assertion path forces save-state tests to duplicate it. Extract the primitive; let both consumers call it.

This is not a "common infrastructure for its own sake" rule — it's a determinism rule: two harnesses with two run-loops produce two stories, and reconciling them is its own bug surface.
