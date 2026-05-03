# Build

`build.zig` shape for a cycle-accurate emulator. Lean reference; not exhaustive Zig build-system theory.

## Module graph: headless core + downstream artifacts

The load-bearing invariant: **the core is headless** — it has no dependencies on a renderer, audio output, wall-clock, or input device. Tests, frontends, debuggers, and analysis tools all consume the core; the core consumes none of them. Adding any frontend/wall-clock dependency to the core breaks the headless-test invariant; resist the temptation.

Beyond that invariant, the artifact graph varies:

- **Core library** — the deterministic emulation. For systems with **multiple clock domains** (e.g., the SNES SPC700 audio coprocessor running asynchronously to the main 65816), the core itself is naturally a module graph: a top-level core module that imports per-domain sub-modules (main CPU, audio coprocessor, DSP, cartridge coprocessors). A single-root core module works for systems with a single SoC (Game Boy, NES); multi-domain systems will want per-domain sub-modules under whatever root file the core is rooted at.
- **Downstream artifacts** — open-ended. Frontends (GUI, TUI, in-process debugger), tools (headless trace dumper, ROM analyzer, save-state dumper), the test-ROM runner, optionally wasm or embedded builds. Each declares its own dependency on the core. The skill doesn't prescribe "one frontend" — ship as many downstream artifacts as the project needs.

The illustrative shape below shows a single-domain core plus one frontend exe; multi-domain cores compose more sub-modules into `core_mod`, and additional downstream artifacts follow the same `addImport("core", core_mod)` pattern.

**All `b.path(...)` arguments below are illustrative.** Substitute whichever paths fit your project's layout — flat (`core.zig` at root), conventional `src/`, multi-package `pkg/<name>/`, or anything else. The pattern is the module-graph shape, not the directory tree.

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    // --- Core library (headless) ---
    // For multi-domain systems (e.g. SNES), this module's root file
    // imports per-domain sub-modules rather than holding all state
    // directly.
    const core_mod = b.createModule(.{
        .root_source_file = b.path("<path/to/core/root>.zig"),
        .target = target,
        .optimize = optimize,
    });

    const core_lib = b.addLibrary(.{
        .name = "<emu>_core",
        .root_module = core_mod,
        .linkage = .static,
    });
    b.installArtifact(core_lib);

    // --- Frontend exe (one of potentially several downstream artifacts) ---
    const frontend_mod = b.createModule(.{
        .root_source_file = b.path("<path/to/frontend/main>.zig"),
        .target = target,
        .optimize = optimize,
    });
    frontend_mod.addImport("core", core_mod);
    // ... add your renderer / audio / input imports here ...

    const frontend_exe = b.addExecutable(.{
        .name = "<emu>",
        .root_module = frontend_mod,
    });
    b.installArtifact(frontend_exe);

    // Additional downstream artifacts (debugger exe, headless trace
    // dumper, wasm build, test-ROM runner, ...) are added with the
    // same shape: a module that imports core_mod, then addExecutable
    // / addLibrary as appropriate.

    // ... test wiring (next section) ...
}
```

## Test wiring

Tests are first-class build steps. The standard shape:

```zig
    // --- Unit tests against the core ---
    const core_tests = b.addTest(.{ .root_module = core_mod });
    const run_core_tests = b.addRunArtifact(core_tests);

    const test_step = b.step("test", "Run core unit tests");
    test_step.dependOn(&run_core_tests.step);

    // --- Test-ROM integration tests ---
    const test_rom_runner = b.addExecutable(.{
        .name = "test-rom-runner",
        .root_module = b.createModule(.{
            .root_source_file = b.path("<path/to/test-rom-runner>.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    test_rom_runner.root_module.addImport("core", core_mod);

    // One addRunArtifact per test-ROM suite, each with its own
    // --suite and --timeout-cycles args (per-suite budget set by
    // the consuming repo, not the skill).
    const suite_step = b.addRunArtifact(test_rom_runner);
    suite_step.addArg("--suite");
    suite_step.addArg("<suite-name>");
    test_step.dependOn(&suite_step.step);
```

`test_rom_runner` is a Zig program that loads test ROMs, runs them through the core, and exits non-zero on failure. A single `zig build test` runs unit tests plus every wired suite.

## Feature flags

Use `b.addOptions()` and `addImport("build_options", ...)` for compile-time feature flags. The pattern:

```zig
    const opts = b.addOptions();
    const my_flag = b.option(
        []const u8,
        "my-flag",
        "Description.",
    ) orelse "default";
    opts.addOption([]const u8, "my_flag", my_flag);

    core_mod.addImport("build_options", opts.createModule());
```

Code reads `@import("build_options").my_flag` and `switch`es on it at compile time. Run with `zig build -Dmy-flag=value`.

This is the typical mechanism for **compile-time** gating of decision #6 (fidelity scope). For runtime gating — where the user picks the revision via CLI flag — don't use `b.addOptions`; use a field on the console struct.

## Build modes

The standard Zig modes (`Debug`, `ReleaseSafe`, `ReleaseFast`, `ReleaseSmall`) all matter for an emulator:

- **`Debug`** — tests, dev. Bounds checks on; safe arithmetic; slow.
- **`ReleaseSafe`** — useful default for shipping to friends-and-family. Bounds checks still on; perf within ~30% of `ReleaseFast`.
- **`ReleaseFast`** — for benchmarks and "is the dispatch loop actually fast" testing. Bounds checks off.
- **`ReleaseSmall`** — useful only if you're shipping to wasm or embedded.

Don't enable `ReleaseFast` by default for shipped builds; the slight perf gain rarely outweighs the safety loss.

## Pinning the Zig version

Pin the Zig version somehow (Nix flake, asdf, mise, manual instructions in README) so dev / pre-commit / CI all run the same compiler. The skill assumes Zig 0.16 — drift between local and CI is the most common source of "works on my machine" emulator-build issues. Pick whichever toolchain-pinning approach matches your habits.

## Submodule init for test ROMs

The first `zig build test` must initialize submodules if they're missing. Either:

- **Document it**: README says "run `git submodule update --init` after cloning." Simple; relies on user discipline.
- **Automate it**: a `b.addSystemCommand(&.{"git", "submodule", "update", "--init"})` step that test_step depends on. Slightly invasive; consult before adopting.

Default: document. Automation can be added later if it becomes friction.

## Cross-references

- [testing.md](./testing.md) — what each test step actually tests.
- [polymorphism.md](./polymorphism.md) — `comptime Bus: type` shapes how the core module is structured.
