# Polymorphism

Choosing between tagged union, vtable, and `comptime` monomorphization for the recurring polymorphism boundaries in an emulator.

## The decision matrix

| Boundary | Cardinality | Open or closed? | Recommended shape |
|---|---|---|---|
| **Mappers / cartridges** | NES ~250, Game Boy ~10, SNES ~30 | Closed historical set — no new ones will be invented | Tagged `union(enum)` with `inline else` dispatch |
| **CPU↔Bus** (per CPU in the system) | 1 per concrete bus type per CPU (single-CPU targets have one; multi-CPU targets have one per CPU) | Closed at compile time within a build | `comptime Bus: type` (monomorphized) |
| **Frontend** (renderer, audio, input) | Open-ended | Open — users may add new ones | Vtable (function-pointer struct) |

The temptation to use a vtable everywhere because "polymorphism is polymorphism" leaves performance on the table for the hot-path closed-set cases (mappers, bus). The temptation to use tagged unions everywhere creates impossible-to-extend frontends. **Match the shape to the openness of the set.**

## Tagged `union(enum)` with `inline else`

For mappers, the tagged union with `inline else` lets the compiler specialize the dispatch per active mapper while preserving the closed-set guarantee at the type system level.

```zig
pub const Mapper = union(enum) {
    // One variant per supported mapper. Each variant struct
    // implements read/write (and any mapper-specific hooks).
    rom_only: RomOnly,
    // ...

    pub fn read(self: *Mapper, addr: u16) u8 {
        switch (self.*) {
            inline else => |*m| return m.read(addr),
        }
    }
    // pub fn write similarly.
};
```

`inline else` instantiates one specialized arm per active variant. The compiler can inline the call into each arm — the resulting code is comparable to a bare function call into the chosen variant's `read` once the runtime tag is known.

When slicing mapper work for `/to-issues`, slice **register-by-register**, not mapper-by-mapper. One slice per mapper register is the right granularity; "implement <some-mapper>" is too coarse.

## `comptime Bus: type` for CPU↔Bus

The CPU is parameterized over the bus type at compile time. There's typically only one concrete bus per build, so monomorphization wins outright.

```zig
pub fn Cpu(comptime Bus: type) type {
    return struct {
        // ... CPU state (PC, SP, register file, flags) ...

        pub fn step(self: *@This(), bus: *Bus) void {
            const op = bus.read(self.pc);
            // ... dispatch ...
        }
    };
}

// Usage: cpu: Cpu(MyBus) and pass a &my_bus into step.
```

Trade-off: testability. With `comptime Bus`, mocking the bus means defining a `MockBus` type with the same surface. That's slightly more friction than passing a vtable, but the duck-typing structural check Zig performs is enough for tests — `MockBus` doesn't need a formal interface declaration.

If you decide testability outweighs the perf gain, use a vtable. But for cycle-accurate work, the bus is on the hottest of paths and should be monomorphized.

### Multi-CPU systems

For systems with more than one programmable CPU (SNES main 65816 + SPC700; cartridge coprocessors like SuperFX or SA-1), each CPU is its own `Cpu(<ItsBus>)` instantiation with its own bus type. The `comptime Bus: type` pattern doesn't change — it just gets applied N times.

What `comptime Bus` does **not** address is **inter-CPU coordination** — when CPU A advances vs CPU B, how they synchronize on inter-CPU communication, how the save-state captures consistent state across both. That's a separate hard-to-reverse decision (catch-up scheduler vs cycle-locked stepping vs coroutine-based) and is decision #7 in [SKILL.md](./SKILL.md). Walk it via `/grill-with-docs` before instantiating the second `Cpu`.

## Vtable (function-pointer struct) for frontends

Frontends are open: renderer / audio / input implementations can be swapped or added at any time. Vtable is the right shape.

```zig
pub const Renderer = struct {
    ctx: *anyopaque,
    vtable: *const Vtable,

    pub const Vtable = struct {
        present: *const fn (ctx: *anyopaque, fb: []const u32) void,
        deinit: *const fn (ctx: *anyopaque) void,
    };

    pub fn present(self: Renderer, fb: []const u32) void {
        self.vtable.present(self.ctx, fb);
    }
    // pub fn deinit similarly.
};
```

Each concrete renderer provides a `fn renderer(self: *Self) Renderer` that supplies its `ctx` + a static `Vtable` whose function pointers `@ptrCast(@alignCast(ctx))` back to `*Self`.

Vtables aren't on the hot path (per-frame `present` calls are cheap), so the indirection is fine.

## Cross-references

- [dispatch.md](./dispatch.md) — opcode dispatch shape that calls into the bus type chosen here.
- [packed-structs.md](./packed-structs.md) — most polymorphic register reads/writes flow through the mapper or bus boundary.
- [testing.md](./testing.md) — test discipline this skill defers to (test ROMs, determinism, save-state round-trip).
