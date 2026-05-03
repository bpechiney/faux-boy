# Testing

Test discipline for cycle-accurate emulators. Drives what `/tdd` consumes; encodes the determinism and save-state rules that are non-negotiable.

`/tdd` runs the red-green-refactor loop. This file says **what to test against** and **how to wire each kind of test**. It does not parallel `/tdd`'s loop discipline.

## The test-ROM tier

Test ROMs are the gold standard for emulator correctness. Each suite has a deterministic pass/fail signal of one of three shapes:

- **Serial-port output** — the ROM prints `Passed` / `Failed` to a fake serial device; the harness scans the captured bytes after a cycle budget.
- **Register magic-pattern** — at completion, the CPU registers hold a sentinel value (e.g. a Fibonacci sequence); the harness asserts on register state.
- **Framebuffer hash** — the harness captures the framebuffer at a specified cycle count and compares against a vendored expected hash.

Per-system suite-to-shape mappings, vendoring policy, and signal-extraction conventions belong in the consuming emulator repo's docs (typically `docs/test-roms.md` or similar). Build wiring lives in [build.md](./build.md). Use a per-suite cycle timeout so a hang is a failure, not a wedged CI run.

## Determinism — non-negotiable

Re-running the same ROM with the same input and the same starting state must produce **bit-identical** internal state at every cycle.

Concrete rule: the test harness runs each ROM twice, captures the full console state at frame 60, and asserts byte-for-byte equality. This test exists in every emudev repo from day one.

```zig
test "determinism: <rom> boot" {
    var c1 = try Console.init(rom);
    defer c1.deinit();
    for (0..60) |_| c1.runFrame();

    var c2 = try Console.init(rom);
    defer c2.deinit();
    for (0..60) |_| c2.runFrame();

    try std.testing.expectEqualDeep(c1.snapshotBytes(), c2.snapshotBytes());
}
```

Common determinism violations to watch for:

- Wall-clock reads in the core. Use simulated cycle counters, not `std.time.milliTimestamp()`.
- `std.crypto.random` or any unseeded RNG.
- Pointer-address-dependent behavior (e.g., hashing on `@intFromPtr`).
- Iteration order over `std.AutoHashMap` (stable per-build, but not stable across allocator differences — prefer arrays or `ArrayHashMap`).

## Save-state round-trip — non-negotiable

Save-state-and-restore must be **bit-identical**. This test also exists in every emudev repo from M0.

```zig
test "save-state round trip: arbitrary mid-frame" {
    const allocator = std.testing.allocator;

    var c1 = try Console.init(rom);
    defer c1.deinit();
    for (0..1234) |_| c1.step(); // arbitrary mid-cycle

    const blob = try c1.serialize(allocator);
    defer allocator.free(blob);

    var c2 = try Console.init(rom);
    defer c2.deinit();
    try c2.deserialize(blob);

    try std.testing.expectEqualDeep(c1.snapshotBytes(), c2.snapshotBytes());
}
```

The `serialize(allocator) ![]u8` / `deserialize([]const u8) !void` shape sidesteps the Zig-stdlib churn around `std.ArrayList` and `std.io` / `std.Io`. Pick whichever stream type matches the current Zig release in your concrete `Console.serialize` / `Console.deserialize` implementations; the round-trip *test* only needs to round-trip a byte blob.

### Field-per-slice discipline

When implementing the save-state schema, slice **one schema field per slice**, not "the whole save-state at once." Each slice:

1. Adds the field to the serialize/deserialize functions.
2. Adds a round-trip test that mid-cycle modifies the field, serializes, deserializes into a fresh console, asserts the field round-trips.
3. Confirms the test passes.
4. Lands.

This catches schema bugs (off-by-one in length-prefixed arrays, wrong byte order, forgotten fields) at single-field granularity instead of "round-trip is broken for some reason in this 4KB blob."

### Versioning

Save-state schema is versioned from day one. The schema header carries `magic`, `version`, and `revision_tag` (the active fidelity-scope revision per decision #6). Migration discipline is decision #4 — typical answer is an explicit ladder where each version-N reader knows how to read version-N-minus-1, with a fallback "this save is too old" error.

Don't decide migration discipline post-hoc — bumping a version without a migration breaks every user save.

## Golden traces

For mid-build verification of cycle-exact paths (specifically the CPU during opcode bring-up), capture **golden traces**: a ROM-driven log of every register of the CPU under test, plus cycle count per instruction. Diff against a known-good reference emulator's trace for the same ROM. The first divergence tells you exactly which opcode mis-implements something. For multi-CPU designs, capture a separate trace per CPU.

This is heavy; reserve for CPU bring-up and cycle-exact PPU paths, not as a default test.

## Frontends and headless

The core must be **headless-capable**. Tests do not require a window or audio device. The frontend is a separate artifact that consumes the core's framebuffer / audio buffer. Tests run against the core directly.

Determinism + headless together mean every test is `zig build test` on a CI runner with no graphical environment. No `xvfb`, no headless-Chrome equivalents. This is the single biggest win the headless-from-day-one decision buys.

## Cross-references

- `/tdd` — drives the red-green-refactor loop using the categories above.
- [build.md](./build.md) — how each test category is wired as a `zig build` step.
- [comments.md](./comments.md) — tests cite the suite they verify via `TEST[<suite>-<test-id>]` tags.
