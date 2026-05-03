# Save-State Review Checklist

This file is the **aggregator**, not the source. Every component checklist's §5 (or equivalent "implicit state" section) is the source of truth for *what* must round-trip:

- `cpu_review.md` §5 — mid-instruction state, NMI latch, IRQ line state, DMA progress.
- `ppu_review.md` §5 — `w` toggle, `t`/`v`/fine-X registers, OAM evaluation state, frame-counter for odd-dot skip.
- `apu_review.md` §7 — frame counter sub-state, per-channel envelope/length/duty/LFSR, DMC byte counters and IRQ flag, resampler buffer.
- `mapper_review.md` §5 — bank registers, IRQ counter latches, RTC sub-second, GSU cache, SA-1 register file.
- `bus_review.md` §5 — open-bus latches, in-progress DMA/HDMA state.

**Walk those sections by reference. Do not enumerate fields again here.** Reviewers cross-reference; they don't re-walk. The questions in this file are *cross-cutting* — versioning, endianness, save-during-execution hazard, round-trip determinism.

Out of scope: **compression and format choice**. Like audio resampling, those are design-review territory. This file reviews whether the implementation of whatever format was chosen is correct (deterministic, versioned, endian-explicit, complete).

---

## 1. Save-during-execution hazard

Save-state captured while a component is mid-step is the single most-shipped class of save-state bug. Catch-up scheduler architectures surface it most often, but it is broader than one architecture — any save hook that fires from inside a component's step function has it.

**Question:** When is the save-state callback allowed to fire? Documented save points should be at component-aligned boundaries (CPU instruction boundary, PPU scanline boundary, scheduler quiescent point). A callback that fires from arbitrary user input — keyboard, network, timer — must defer to the next quiescent point or capture all in-flight component state.

**Question:** If the implementation captures mid-step state, does the schema include the necessary continuation cursors? CPU mid-instruction T-state index. Catch-up scheduler "this component is halfway through processing event T+3" markers. APU resampler fractional-position. PPU mid-scanline dot index.

**Question:** If the implementation defers to quiescent points, is the deferral *visible* — i.e., can the user observe that "save now" actually saved a few microseconds later? Silent deferral is an invariant violation if the save name encodes a timestamp the user expects to match.

**Question:** Loading a save taken mid-step on a *different* architecture-shape implementation produces what behaviour? E.g., a save from a per-cycle CPU loaded into a micro-op-table CPU — does the schema fail loudly (preferred) or silently miscount cycles? Document this.

---

## 2. Versioning

**Question:** File-header version present, with a documented bump policy (every schema-affecting change increments)?

**Question:** Forward-compat strategy: when a newer build adds a field, what does an older build do on load? Three valid policies:
- **Reject older saves** — fine, but make sure the error message is informative ("save is version 3, build supports version 4 only").
- **Field-presence flags** — newer build can read older save by treating new fields as defaults.
- **Migration functions** — explicit `migrate_v3_to_v4(state) -> state`. Most robust; most maintenance overhead.

**Question:** Backward-compat: do we support loading saves *from* newer builds in older builds? Usually no, and that's fine — but the policy should be explicit and the failure should be loud (not a silent half-load that produces a corrupt running state).

**Question:** A schema-affecting change without a version bump is the bug to flag. Look for: new field added in a `pub fn serialize` without a corresponding version-byte change.

---

## 3. Endianness

**Question:** Is endianness **explicit** in the serialization code, or implicit via `@bitCast` / `mem.bytesAsSlice`?

`@bitCast` of a multi-byte field on a big-endian host produces a different byte sequence than on a little-endian host. The save file is no longer portable. The fix is to use `std.mem.writeInt(.little, …)` (or `.big`) at every multi-byte field. There is no exception.

**Question:** For struct-packing-via-`@bitCast`: the struct memory layout is *not* a stable serialization format. Even on the same host, Zig may reorder fields, pad differently, or change widths in a future release. Per-field explicit serialization is the only safe form.

**Question:** Endian-mark in the file header — required for forensics even if the format is "always little-endian" by policy. A 4-byte magic + 1-byte version + 1-byte endian mark is the canonical shape.

---

## 4. Pointer serialization (the "no addresses" rule)

**Question:** No raw pointers serialized as integer addresses?

A tagged-union mapper field that contains a pointer to its backing data, written as `@intFromPtr(self.mapper_state)`, will not deserialize on a different process / build / OS — the address is meaningless on load. Even within a single process, ASLR makes this nondeterministic.

**Question:** When serializing references between components, use **stable IDs**: channel index, mapper register index, voice number — never pointers.

**Question:** When serializing buffers, serialize the bytes directly (with explicit length prefix). When serializing a reference *into* a buffer (e.g., DMC's "current sample address" within ROM), serialize the offset, not the absolute pointer.

**Question:** Slices that point into ROM image: serialize as offset+length. On load, validate the offset against the loaded ROM's size — corrupt save files in the wild may have offsets past EOF.

---

## 5. Round-trip determinism — the empirical test

The cleanest empirical test for save-state completeness is the **round-trip determinism check**:

1. Run from a known initial state for N master cycles.
2. Capture frame buffer hash + audio buffer hash + system state hash at cycle N.
3. From the same initial state, save at cycle N/2, immediately restore, then run the remaining N/2 cycles.
4. Capture hashes at cycle N again.
5. Hashes must be byte-identical.

**Question:** Does the test infrastructure include this round-trip check, run on a representative ROM at multiple save points (boot, mid-frame, vblank entry, mid-DMA, mid-NMI handler)?

**Question:** Hash function is deterministic (SHA-256, Blake3, FNV-1a — all fine; not a hash that uses host-RNG)?

**Question:** Cross-references with `references/test_roms.md` — golden-frame regression suites are the building blocks for this test. A frame-hash-at-cycle-N test embedded in a known ROM is the most rigorous form.

A save-state implementation without a round-trip determinism test is shipping on faith. Note this in the review output as a yellow flag, even if no specific bug is found.

---

## 6. Determinism intersection (methodology rule 4)

Save-state is a determinism test surface. Bugs that previously hid in normal play surface immediately:

- Host-RNG-seeded uninitialised RAM → save-then-load shows different RAM than continuous run.
- `std.AutoHashMap` iteration order changing post-load (different bucket count after deserialize).
- Time-based seeding read in advance and saved, then re-read on load → drift.

Cross-reference each component's determinism section. Save-state is where rule-4 violations manifest as user-visible bugs.

---

## 7. Validation on load

**Question:** Magic bytes + version checked, with informative error on mismatch (not a panic, not silent garbage)?

**Question:** Field bounds validated? A corrupt save with `cpu.PC = 0xFFFFFFFF` should produce a load error, not a panic on first instruction fetch. A reasonable hardening level: every multi-byte numeric field's value must be in its hardware-valid range. Not exhaustive; defensive enough to catch obvious corruption.

**Question:** Per-component checksums in the save file, or a single file-level checksum? Either is fine; without either, partially-corrupted saves are debugging hell.

**Question:** `unreachable` / panic on unrecognised tagged-union variant during load: hard error appropriate for "the save is from a future build with a new mapper variant." Document the user-visible behaviour.

---

## 8. Compatibility documentation

The save-state contract is a *user-facing* compatibility surface for any emulator that ships saves. Review the documentation alongside the code:

**Question:** Is there a documented schema (even a comment in the source) listing every field, its size, and its semantics?

**Question:** Are version bumps documented in a CHANGELOG (or equivalent), with the migration policy stated?

**Question:** Are *expected* breakage points documented? "Saves from version 3 cannot be loaded in version 4 because the mapper layout changed" is a fine answer if it's *documented*.

---

## 9. Test-ROM correspondence (methodology rule 1)

Save-state test ROMs are sparse — most testing is emulator-internal round-trip suites. See `references/test_roms.md` "Save-state tests" section for the canonical (and mostly empty) list, and the policy on tests-that-don't-exist.

| Change touches... | Re-run at minimum |
|---|---|
| Schema versioning | round-trip test on every test ROM at multiple save points |
| Pointer serialization | full save → restart-emulator-process → load test (catches address-dependence) |
| Endianness | save on little-endian host, load on big-endian host (or fake via test) |
| Mid-execution save | round-trip with save points at each cycle modulo small N (e.g., every 7th master cycle) |

A save-state change that ships without round-trip testing is the most expensive class of bug to catch later — note the gap explicitly.
