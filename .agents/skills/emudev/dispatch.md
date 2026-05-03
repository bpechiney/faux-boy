# Dispatch

CPU opcode dispatch in cycle-accurate emulators in Zig 0.16.

## The recommended pattern: labeled `switch` with `continue :dispatch`

Zig 0.16's labeled `switch` form is the recommended dispatch pattern for opcode loops. Zig's own tokenizer measured a +13% throughput gain converting from a giant `switch` to this form. Of ~12 surveyed Zig-language emulators in late 2025, **zero** used it — the pattern is non-default. Greenfield emulator projects should adopt it from day one.

```zig
// Op is an enum(u8) of every opcode for the target CPU.
// Cpu is parameterized over Bus (see polymorphism.md), so step takes *Bus.
pub fn step(cpu: *Cpu, bus: *Bus) void {
    var op: Op = @enumFromInt(bus.read(cpu.pc));
    cpu.pc +%= 1;

    dispatch: switch (op) {
        .some_opcode => {
            // ... operand reads, register updates, cpu.tick(N) ...
            return;
        },
        .opcode_that_chains => {
            // ... do the work ...
            // Tail-chain into the next instruction without leaving
            // the dispatch frame.
            op = @enumFromInt(bus.read(cpu.pc));
            cpu.pc +%= 1;
            continue :dispatch op;
        },
        // ... one arm per opcode ...
    }
}
```

Key points:

- `dispatch:` labels the `switch`; `continue :dispatch op;` re-enters with a new tag value without unwinding the frame, letting the compiler specialize the indirect jump per arm.
- Tail-chaining is most useful for opcode sequences where one opcode's body wants to fall through to the next opcode's dispatch without unwinding (e.g., when a single byte of work belongs to a multi-step instruction whose later steps share dispatch infrastructure).

## When to use a function-pointer table instead

Function-pointer tables trade compiler specialization (labeled `switch` wins on the hot path) for runtime patchability (tables win when you need to swap an opcode's behavior at runtime, e.g., debugger trap insertion).

**Recommendation**: labeled `switch` for the base opcode table. Reach for a function-pointer table only when runtime patching is the actual need — and even then prefer a `comptime` parameter selecting between "trapping" and "non-trapping" dispatch loops, instantiated as separate concrete functions.

## `@branchHint` for hot/cold paths

Zig 0.16 supports `@branchHint(.likely)` / `.unlikely` / `.cold`. Use sparingly — the compiler's heuristics usually beat hand-tuning. The two cases worth hinting: the rarely-taken arm of an interrupt-service branch, and `@panic` / `unreachable` arms in dispatch.

## Cross-references

- [comments.md](./comments.md) — every `case` arm whose timing is non-obvious carries a `REF` or `QUIRK` tag.
- [polymorphism.md](./polymorphism.md) — for the bus type the dispatch operates on.
- [packed-structs.md](./packed-structs.md) — for register reads/writes inside opcode bodies.
