# Comments — citation taxonomy

Generic Zig culture treats comments as a code smell. Emulator code inverts this: **uncited timing-sensitive code is the smell**. Hardware-derived code without a citation is unreviewable — a future reader can't tell whether the magic number came from a doc, a test ROM, a real-hardware capture, or a guess.

## The six tags

Every function or block whose existence is hardware-derived must carry at least one of these tags. The cardinality is six (not seven, not five) — small enough to memorize, large enough to disambiguate the common citation kinds.

### `REF` — external citation

Citation of an authoritative external source — a hardware reference doc, an ADR, or a CPU manual. Almost always co-occurs with `QUIRK` or `HW`.

```zig
// REF: pandocs/oam_dma.html
// "DMA copies 160 bytes from source to OAM. The CPU is effectively
//  halted (only HRAM access works) for 160 cycles."
fn startOamDma(self: *Bus, src: u8) void { ... }
```

Prefer the **verbatim-quote pattern**: paste the cited sentence into the comment block. URL rot doesn't void verbatim quotes — the source text is preserved in the source.

For ADR references, cite the file path (not a numeric ID): `// REF: docs/adr/<NNNN-slug>.md`. Paths are stable; numeric shorthand is less greppable.

**Don't cite**: the Zig stdlib (the agent has stdlib knowledge), common-knowledge CPU concepts (what `ADD A, B` does), or your own code in another file (use module imports and `///` doc comments instead).

### `QUIRK` — non-obvious universal hardware behavior

Real hardware does this; your emulator must reproduce it (subject to the cycle-accuracy tier you committed to). Distinct from `HW` — `QUIRK` is universal across the system; `HW` is revision-specific.

```zig
// QUIRK: HALT-bug. If HALT runs while IME=0 and (IF & IE) != 0, the
// next instruction's first byte is read but PC does not increment —
// the byte is executed twice.
// REF: pandocs/halt.html
```

### `HW` — hardware-revision-specific

Behavior varies between revisions of the same console. Use the **community-canonical revision codename for the system you're targeting** — each system has its own canon (e.g. DMG/MGB/SGB/SGB2/CGB/AGB for Game Boy hardware revisions, 2C02/2C07 for NES PPU variants, 1-CHIP/2-CHIP/3-CHIP for SNES PPU board revisions). The bracketed value is the revision (or set of revisions) the code path applies to.

```zig
// HW[CGB,AGB]: KEY1 register controls CPU clock doubling.
// Not present on DMG/MGB/SGB family.
```

For paths gated at compile-time on the active fidelity scope (decision #6), the `HW` tag annotates the `comptime` branch.

### `TEST` — test ROM exercises this path

Bracketed test name when applicable. Useful for greppable regression context: `rg "TEST\[<suite-prefix>"` across the source tree finds every code path a given test suite exercises.

```zig
// TEST[<suite>-<test-id>]: validates <what>. Make the bracketed name
// match exactly what `rg "TEST\[<suite>"` should find when triaging
// regressions in that suite.
```

### `HACK` — imperfect approximation

Requires **all three** of the following — without all three, it's not a `HACK`, it's bad code. Delete instead of tag:

1. Bracketed game or test name (`HACK[<Game>]`) — what real-world thing forced this.
2. Linked issue (`(#N)`) tracking removal.
3. Stated hardware uncertainty — what we don't yet know that justifies the imperfection.

```zig
// HACK[<Game>] (#N): <what we do that's imperfect> — <why we do it,
// e.g., the game would otherwise hang on a specific path>. Hardware
// uncertainty: <what we don't yet know that would let us fix this
// properly>. Remove when <removal condition, e.g., a specific test
// passes without this hack>.
```

Every active `HACK` is also catalogued in the typed `Hacks` namespace (location declared in `docs/agents/emudev.md`). The inline tag is the breadcrumb at the use site; the namespace is the auditable catalog.

### `TODO` — unverified or missing

Always carries a linked issue. No floating TODOs.

```zig
// TODO(#N): <what's unverified or missing> — <how to verify or
// what's needed to finish>.
```

## Naming hygiene: `QUIRK` vs fidelity scope

The `QUIRK` tag annotates **universal** non-obvious hardware behavior (mid-instruction interrupt edge cases, sprite-priority resolution oddities, mid-frame palette/OAM write timing, etc.). These behaviors are present on every revision of the system; whether you reproduce them is dictated by your cycle-accuracy tier (decision #3).

The phrase **"fidelity scope"** (decision #6) refers to the ADR-level scope choice — *which revisions, regions, peripherals, boot ROMs, and analog characteristics* this emulator reproduces at all. That's not a `QUIRK` tag — it's a `comptime` configuration, a revision flag, an ADR.

Two distinct concepts, two distinct names. Don't tag a fidelity-scope decision with `QUIRK`.

## Proximity rule: per-function-or-block

Every function or block whose existence is hardware-derived carries at least one tag. **Not per-N-lines** — that forces noise into mechanical regions (a long opcode dispatch table doesn't need 20 tags, just one at the top of each opcode's handler if its behavior is non-obvious).

Per-function-or-block matches the unit of decision: emulator code reasons in functions, not line windows.

## Doc comments

Reserve `///` doc comments for the public API of a module (functions exported across module boundaries). Inline `//` comments carry the citation tags. Don't double up — a `pub fn` with both a `///` summary and an inline `// REF:` is fine, but the citation tag never lives in the `///` block.

## Grep recipes

Useful when reviewing emulator code or auditing the citation surface.

```bash
# Set these to match your project's layout.
SRC=src                                  # source root
HW_DIRS="$SRC/cpu $SRC/ppu $SRC/apu $SRC/bus $SRC/mappers $SRC/cart"

# Hardware-derived files with no citation tag at all.
# The "([[(][^:]*)?" admits the bracketed/parenthesized forms
# (HW[<rev>]: / TEST[<id>]: / HACK[<game>] (#N): / TODO(#N):)
# alongside the bare REF: / QUIRK: forms.
rg --files $HW_DIRS | xargs -I{} sh -c '
  grep -lE "(REF|QUIRK|HW|TEST|HACK|TODO)([[(][^:]*)?:" "{}" > /dev/null || echo "{}"
'

# Untracked HACKs (no issue link)
rg "HACK\[" $SRC | grep -vE "#[0-9]+"

# Floating TODOs (no issue link)
rg "TODO" $SRC | grep -vE "TODO\(#[0-9]+\)"

# All hardware revisions touched
rg -or '$1' 'HW\[([^]]+)\]' $SRC | sort -u

# All games / test ROMs referenced
rg "(HACK|TEST)\[([^]]+)\]" $SRC -or '$2' | sort -u

# Density check: tag count vs hardware-derived file count.
# Same regex shape as the missing-citation check above.
echo "tags:";  rg -c "(REF|QUIRK|HW|TEST|HACK|TODO)([[(][^:]*)?:" $SRC
echo "files:"; rg --files $HW_DIRS | wc -l
```

These are convention checks, not enforcement gates. Wiring any of them into CI is a per-repo decision (see `build.md`).
