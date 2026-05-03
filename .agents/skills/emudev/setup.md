# Emudev setup — lazy creation

Run this when emudev is invoked in a repo without `docs/agents/emudev.md`. Idempotent and re-runnable.

## What this writes

1. **`docs/agents/emudev.md`** — system marker + emudev-specific config the skill reads at session start.
2. **`AGENTS.md` or `CLAUDE.md`** — adds an `### Emudev` section to the existing `## Agent skills` block (created by `/setup-matt-pocock-skills`).

It does **not** scaffold anything else. The ADR skeleton, `Hacks` namespace stub, and test-ROM directory are deferred to when you actually need them — the YAGNI cost of speculative scaffolding outweighs the convenience.

## Preconditions

- The repo should already have a `## Agent skills` block in `AGENTS.md` or `CLAUDE.md` from `/setup-matt-pocock-skills`. If the block is missing, prompt the user to run `/setup-matt-pocock-skills` first. Don't create the parent block here.
- The repo's primary language should be Zig 0.16-ish. If `build.zig.zon` is missing, ask the user to confirm before proceeding.

## Interview

Ask one question at a time. Don't dump the form.

### 1. System

> Which retro console does this emulator emulate? `gameboy`, `nes`, or `snes`?

The valid set is exactly those three. If the user names something else (CHIP-8, Atari 2600, Genesis, etc.), stop and tell them: this skill targets Game Boy / NES / SNES only; other systems would require extending the skill first.

### 2. Cycle-accuracy tier

> What cycle-accuracy tier are you committing to: `instruction`, `m-cycle`, or `t-state`?

This is hard to upgrade later — picking M-cycle then trying to get T-state accuracy means rewriting the CPU loop. Pick the highest tier you intend to ship.

If user is unsure, recommend running `/grill-with-docs` against decision #3 first, then come back.

Common starting points (not prescriptions — verify against the test suites you care about):
- Game Boy → `m-cycle`
- NES → `m-cycle` or `t-state` depending on whether pixel-precise PPU is in scope
- SNES → `t-state`

### 3. Test-ROM submodule root

> Where will test-ROM submodules live? Default: `tests/test-roms/`.

The directory does not need to exist yet. The path is recorded; the first test-ROM commit creates the dir.

### 4. `Hacks` namespace location

> Where will the typed `Hacks` namespace live? Default: `src/hacks.zig` (re-exported via the core module).

The file does not need to exist yet. First `HACK` you tag will need a corresponding namespace entry — that's when the file gets created.

The Zig version is **not** asked here — `build.zig.zon`'s `minimum_zig_version` is the single source of truth, and SKILL.md's session-start sequence reads it from there. Persisting a separate `zig_version` field in `docs/agents/emudev.md` would create dead data that drifts.

## Write `docs/agents/emudev.md`

```markdown
# Emudev config

Consumed by the `/emudev` skill at session start. Edit by hand if needed; re-running emudev's lazy creation will preserve any unrecognized fields.

system: gameboy
cycle_accuracy_tier: m-cycle
test_rom_root: tests/test-roms/
hacks_path: src/hacks.zig
```

## Amend `AGENTS.md` / `CLAUDE.md`

Add an `### Emudev` subsection to the existing `## Agent skills` block. Use whichever of `AGENTS.md` or `CLAUDE.md` the existing block lives in (don't pick one if both exist; mirror the convention `/setup-matt-pocock-skills` chose).

Example, sibling to the existing Issue tracker / Triage labels / Domain docs subsections:

```markdown
### Emudev

System: gameboy. Cycle-accuracy tier: m-cycle. Test-ROM submodules under `tests/test-roms/`. Typed `Hacks` namespace at `src/hacks.zig`. See `docs/agents/emudev.md`.
```

If the `### Emudev` subsection already exists, **edit it in place**. Don't append a duplicate.

## Idempotency

If `docs/agents/emudev.md` already exists when emudev is invoked:

1. Read it.
2. Show the user the current values.
3. Ask which (if any) to change. Don't re-ask all four questions.
4. Write the file back with edited values, preserving any fields you don't recognize (the user may have hand-added something).
5. Update the `### Emudev` subsection in `AGENTS.md`/`CLAUDE.md` to match.

Never blindly overwrite. Never delete fields the skill doesn't recognize.

## Post-setup pointer

After lazy creation completes, suggest the next step:

> Setup complete. Before implementing significant work, consider running `/grill-with-docs` against the seven load-bearing decisions named in `/emudev`'s SKILL.md (#7 only applies to multi-CPU designs). ADRs landed early in greenfield work cost less than ADRs landed late.
