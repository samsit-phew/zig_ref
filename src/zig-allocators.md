# Memory Allocators in Zig 0.16

## Table of Contents

- [Chapter 1: Introduction to Zig Allocator Model](#chapter-1-introduction-to-zig-allocator-model)
- [Chapter 2: The Allocator Interface](#chapter-2-the-allocator-interface)
- [Chapter 3: page_allocator — Direct OS Page Allocation](#chapter-3-page_allocator--direct-os-page-allocation)
- [Chapter 4: heap.ArenaAllocator — Lock-Free and Thread-Safe](#chapter-4-heaparenaallocator--lock-free-and-thread-safe)
- [Chapter 5: heap.FixedBufferAllocator](#chapter-5-heapfixedbufferallocator)
- [Chapter 6: heap.GeneralPurposeAllocator (GPA)](#chapter-6-heapgeneralpurposeallocator-gpa)
- [Chapter 7: heap.ScratchAllocator](#chapter-7-heapscratchallocator)
- [Chapter 8: Custom Allocator Patterns](#chapter-8-custom-allocator-patterns)
- [Chapter 9: Allocator Best Practices](#chapter-9-allocator-best-practices)
- [Chapter 10: Project — A Memory Pool Allocator](#chapter-10-project--a-memory-pool-allocator)

---

## Chapter 1: Introduction to Zig Allocator Model

Zig's approach to memory management is one of its most distinctive features. Unlike C, which requires you to manually call `malloc` and `free`, or Rust, which uses ownership and borrowing, Zig uses an **explicit allocator parameter** model. Every function that needs to allocate memory takes an `std.mem.Allocator` as a parameter, making allocation strategy a conscious, local decision rather than a global one.

The core insight is this: **allocation is not hidden**. When you see a function signature like:

```zig
pub fn parse(allocator: std.mem.Allocator, input: []const u8) !Parsed { ... }
```

you immediately know this function may allocate memory. The caller controls *which* allocator is used, allowing the same function to work with stack buffers, arena allocators, the system allocator, or a custom pool allocator without any code changes inside the function.

### Why This Matters

1. **Testability**: Pass a counting allocator to detect leaks. Pass a failing allocator to test error paths.
2. **Performance**: Use an arena for short-lived work, a pool for fixed-size objects, or the GPA for debug builds.
3. **Determinism**: No hidden global state. Allocation behavior is fully determined by what you pass in.
4. **No GC**: No garbage collector pauses, no reference counting overhead, no hidden costs.

In Zig 0.16, the allocator model has been further refined. The `heap.ArenaAllocator` is now **lock-free and thread-safe by default**, and the old `heap.ThreadSafeAllocator` wrapper has been **removed entirely**. This is a major simplification.

---

## Chapter 2: The Allocator Interface

The `std.mem.Allocator` type is a struct with function pointers:

```zig
pub const Allocator = struct {
    ptr: *anyopaque,
    vtable: *const VTable,

    pub const VTable = struct {
        alloc: *const fn (ptr: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8,
        resize: *const fn (ptr: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool,
        free: *const fn (ptr: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void,
    };
};
```

### Core Operations

**alloc** — Allocates `len` bytes with the given alignment. Returns `null` on failure (out of memory).

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.getStdOut().writer();

    // Allocate 100 bytes
    const ptr = try allocator.alloc(u8, 100);
    defer allocator.free(ptr);

    try stdout.print("Allocated {} bytes at {*}\n", .{ptr.len, ptr});
}
```

**resize** — Grows or shrinks an existing allocation. Returns `true` on success, `false` if the allocation was moved (in which case the caller should copy data from the old location if needed, though in practice Zig's higher-level wrappers handle this).

**free** — Releases memory back to the allocator.

### Convenience Methods

The `Allocator` struct also provides high-level convenience methods:

```zig
// alloc(Type, count) — allocate an array of `count` items of `Type`
const slice = try allocator.alloc(u32, 10);
defer allocator.free(slice);

// create(Type) — allocate a single value, returns a pointer
const value = try allocator.create(u32);
defer allocator.destroy(value);
value.* = 42;
```

### Sentinel-Terminated Allocation

```zig
// Allocate a null-terminated string
const s = try allocator.dupeZ(u8, "hello");
defer allocator.free(s);
// s is [*:0]const u8
```

---

## Chapter 3: page_allocator — Direct OS Page Allocation

The `std.heap.page_allocator` allocates memory directly from the operating system. Each allocation requests whole memory pages (typically 4096 bytes on x86-64). This is the most basic allocator — it never frees memory back to the OS (though the OS will reclaim it when the process exits).

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const stdout = init.io.getStdOut().writer();

    // Each allocation is at least one page (4096 bytes typically)
    const memory = try allocator.alloc(u8, 100);
    defer allocator.free(memory);

    try stdout.print("page_allocator: allocated {} bytes (uses full OS pages)\n", .{memory.len});

    // Useful for very large allocations where you want OS-level memory
    const big = try allocator.alloc(u8, 1024 * 1024); // 1 MB
    defer allocator.free(big);
    try stdout.print("page_allocator: allocated 1 MB chunk\n", .{});
}
```

### When to Use page_allocator

- You need memory mapped at page-aligned boundaries (e.g., for DMA, shared memory).
- You are allocating very large buffers where the overhead of page-level granularity is acceptable.
- You want the simplest possible allocation with no bookkeeping.

**Never use page_allocator for small allocations** — each 100-byte allocation consumes an entire 4KB page, wasting over 97% of the memory.

---

## Chapter 4: heap.ArenaAllocator — Lock-Free and Thread-Safe

In Zig 0.16, `heap.ArenaAllocator` has been dramatically improved. It is now **lock-free and thread-safe by default**. The old `heap.ThreadSafeAllocator` wrapper has been **removed entirely** — it is no longer needed.

`ArenaAllocator` allocates from a backing allocator but never frees individual allocations. Instead, you call `deinit()` once to release everything at once. This makes it ideal for:

- Parsing: allocate all parsed structures, then free them all at once.
- Request handling: allocate per-request data, clean up at end of request.
- Build phases: allocate during compilation/build, release when done.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();
    const stdout = init.io.getStdOut().writer();

    // ArenaAllocator is now lock-free and thread-safe — no ThreadSafeAllocator needed!
    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit();
    const arena_alloc = arena.allocator();

    // All allocations from arena_alloc are freed when arena.deinit() is called
    const names = try arena_alloc.alloc([]const u8, 5);
    names[0] = try arena_alloc.dupe(u8, "Alice");
    names[1] = try arena_alloc.dupe(u8, "Bob");
    names[2] = try arena_alloc.dupe(u8, "Charlie");
    names[3] = try arena_alloc.dupe(u8, "Diana");
    names[4] = try arena_alloc.dupe(u8, "Eve");

    for (names, 0..) |name, i| {
        try stdout.print("  [{}] {}\n", .{i, name});
    }

    // No need to free individual allocations — arena.deinit() handles everything
    try stdout.print("\nArena allocated {} bytes\n", .{arena.queryCapacity() orelse 0});
}
```

### Thread Safety

Because `ArenaAllocator` in 0.16 is lock-free, you can safely share its allocator across threads:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.getStdOut().writer();

    var arena = std.heap.ArenaAllocator.init(allocator);
    defer arena.deinit();
    const arena_alloc = arena.allocator();

    // Safe to pass arena_alloc to multiple threads
    var threads: [4]std.Thread = undefined;
    for (&threads, 0..) |*t, i| {
        t.* = try std.Thread.spawn(.{}, workerFn, .{ arena_alloc, i });
    }
    for (&threads) |t| {
        t.join();
    }

    try stdout.print("All threads completed. Arena capacity: {}\n", .{arena.queryCapacity() orelse 0});
}

fn workerFn(allocator: std.mem.Allocator, id: usize) void {
    // Each thread can safely allocate from the shared arena
    const data = allocator.alloc(u8, 64) catch return;
    // No need to free — arena cleanup handles it
    _ = data;
    _ = id;
}
```

### arena.allocator() Returns a New Allocator Each Time

Note that in 0.16, `arena.allocator()` returns a new `std.mem.Allocator` value each time. Each one is safe to use concurrently. If you need only one thread using the arena, you can call `arena.allocator()` once and reuse it.

---

## Chapter 5: heap.FixedBufferAllocator

`heap.FixedBufferAllocator` allocates from a pre-existing buffer, typically on the stack or in a global. It never calls the OS allocator. When the buffer is full, allocation fails with `OutOfMemory`.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // 1 KB buffer on the stack
    var buffer: [1024]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    // Allocate strings from the stack buffer
    const s1 = try allocator.alloc(u8, 10);
    @memcpy(s1, "Hello FBA!");

    const s2 = try allocator.alloc(u8, 20);
    @memcpy(s2, "Stack allocation rocks!");

    try stdout.print("s1: {s}\n", .{s1});
    try stdout.print("s2: {s}\n", .{s2});

    // Check remaining capacity
    const used = fba.end_index;
    try stdout.print("Used: {}/1024 bytes\n", .{used});

    // Reset the allocator (frees everything)
    fba.reset();
    const used_after_reset = fba.end_index;
    try stdout.print("After reset: {}/1024 bytes\n", .{used_after_reset});
}
```

### Compile-Time Fixed Buffers

Combine with comptime for zero-overhead allocation:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    var buffer: [256]u8 = undefined;
    var fba = std.heap.FixedBufferAllocator.init(&buffer);
    const allocator = fba.allocator();

    // Parse a simple config string using only stack memory
    const config = "host=localhost port=8080 debug=true";
    var it = std.mem.splitSequence(u8, config, " ");
    while (it.next()) |kv| {
        const pair = try allocator.dupe(u8, kv);
        try stdout.print("  config: {s}\n", .{pair});
    }
}
```

### When to Use FixedBufferAllocator

- Embedded systems with no heap.
- Performance-critical sections where heap allocation is too slow.
- Parsing small inputs where stack space suffices.
- Testing with controlled, bounded allocation.

---

## Chapter 6: heap.GeneralPurposeAllocator (GPA)

The GPA is Zig's **debug allocator**. It tracks every allocation, detects double-frees, use-after-free, and memory leaks. It wraps an underlying allocator (defaulting to `page_allocator`).

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Create GPA with default settings
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer {
        // deinit() returns a leak report
        const leaked = gpa.deinit();
        if (leaked == .leak) {
            try stdout.print("WARNING: Memory leaks detected!\n", .{});
        } else {
            try stdout.print("No memory leaks. Clean shutdown.\n", .{});
        }
    }
    const allocator = gpa.allocator();

    // Use it like any other allocator
    const data = try allocator.alloc(u64, 100);
    defer allocator.free(data);

    for (data, 0..) |*item, i| {
        item.* = i * i;
    }

    try stdout.print("GPA: allocated {} u64s\n", .{data.len});
    try stdout.print("  Sum of squares: {}\n", .{
        var sum: u64 = 0;
        for (data) |v| sum += v;
        sum;
    });
}
```

### GPA Configuration Options

The GPA accepts a configuration struct:

```zig
var gpa = std.heap.GeneralPurposeAllocator(.{
    .safety = true,           // Enable safety checks (default: true in debug)
    .never_unmap = false,     // Never return memory to OS (useful for debugging)
    .retain_metadata = false,  // Keep metadata after free (for advanced debugging)
}){};
```

### Detecting Leaks

The key feature: `gpa.deinit()` returns an enum:

```zig
const result = gpa.deinit();
switch (result) {
    .ok => {},                  // Clean
    .leak => {},                // Memory was not freed
}
```

**Always use GPA during development.** Switch to a simpler allocator for release builds if the overhead is measurable.

---

## Chapter 7: heap.ScratchAllocator

`heap.ScratchAllocator` provides a small, fast allocation buffer that can be used for temporary allocations without heap involvement. In Zig 0.16, it is available through `std.heap.scratch_allocator` or by creating one manually.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const allocator = init.allocator;

    // ScratchAllocator is useful for small, temporary allocations
    // It uses a thread-local buffer
    const scratch = std.heap.scratch_allocator;

    // For small temporary allocations, scratch is very fast
    const temp = try scratch.alloc(u8, 64);
    @memcpy(temp, "temporary data here, no heap needed!");

    try stdout.print("Scratch: {s}\n", .{temp});
    // No explicit free needed — scratch resets automatically
}
```

### When ScratchAllocator Shines

- Formatting temporary strings for output.
- Small intermediate buffers in algorithms.
- Any allocation you know is tiny and short-lived.

---

## Chapter 8: Custom Allocator Patterns

One of Zig's most powerful features is the ability to easily wrap or compose allocators.

### Counting Allocator

Wraps any allocator and counts allocations:

```zig
const std = @import("std");

const CountingAllocator = struct {
    inner: std.mem.Allocator,
    alloc_count: u64 = 0,
    free_count: u64 = 0,
    bytes_allocated: u64 = 0,

    fn allocator(self: *CountingAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = allocFn,
                .resize = resizeFn,
                .free = freeFn,
            },
        };
    }

    fn allocFn(ptr: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        const self: *CountingAllocator = @ptrCast(@alignCast(ptr));
        const result = self.inner.vtable.alloc(self.inner.ptr, len, ptr_align, ret_addr);
        if (result != null) {
            self.alloc_count += 1;
            self.bytes_allocated += len;
        }
        return result;
    }

    fn resizeFn(ptr: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        const self: *CountingAllocator = @ptrCast(@alignCast(ptr));
        return self.inner.vtable.resize(self.inner.ptr, buf, buf_align, new_len, ret_addr);
    }

    fn freeFn(ptr: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        const self: *CountingAllocator = @ptrCast(@alignCast(ptr));
        self.free_count += 1;
        self.inner.vtable.free(self.inner.ptr, buf, buf_align, ret_addr);
    }
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();

    var counter = CountingAllocator{ .inner = gpa.allocator() };
    const allocator = counter.allocator();

    // Use the counting allocator
    var items = std.ArrayList(u32).init(allocator);
    defer items.deinit();

    for (0..100) |i| {
        try items.append(@intCast(i));
    }

    try stdout.print("Allocations: {}\n", .{counter.alloc_count});
    try stdout.print("Frees:       {}\n", .{counter.free_count});
    try stdout.print("Bytes alloc: {}\n", .{counter.bytes_allocated});
}
```

### Failing Allocator

An allocator that always fails — perfect for testing error paths:

```zig
const std = @import("std");

const FailingAllocator = struct {
    fn allocator() std.mem.Allocator {
        return .{
            .ptr = undefined,
            .vtable = &.{
                .alloc = allocFail,
                .resize = resizeFail,
                .free = freeNoop,
            },
        };
    }

    fn allocFail(_: *anyopaque, _: usize, _: u8, _: usize) ?[*]u8 {
        return null;
    }
    fn resizeFail(_: *anyopaque, _: []u8, _: u8, _: usize, _: usize) bool {
        return false;
    }
    fn freeNoop(_: *anyopaque, _: []u8, _: u8, _: usize) void {}
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();
    const fail_alloc = FailingAllocator.allocator();

    // This should fail gracefully
    const result = fail_alloc.alloc(u8, 100);
    if (result == null) {
        try stdout.print("FailingAllocator correctly returned null\n", .{});
    }
}
```

---

## Chapter 9: Allocator Best Practices

### Decision Tree

| Scenario | Recommended Allocator |
|----------|----------------------|
| Debug builds | `GeneralPurposeAllocator` |
| Short-lived batch allocations | `ArenaAllocator` |
| Stack-only, no heap | `FixedBufferAllocator` |
| Very large, OS-level buffers | `page_allocator` |
| Small temporary values | `scratch_allocator` |
| Production release | `c_allocator` or GPA with safety off |
| Thread-shared scratch memory | `ArenaAllocator` (thread-safe in 0.16) |

### Rules of Thumb

1. **Always pass allocators as parameters.** Never use a global allocator implicitly.
2. **Use `defer allocator.free(...)` immediately after allocation.** This prevents leaks even on early returns.
3. **Use ArenaAllocator for parsing and request handling.** The batch-free pattern eliminates whole classes of leaks.
4. **Use GPA during development.** The leak detection catches bugs early.
5. **Never store an allocator reference across async boundaries** without understanding its lifetime.
6. **Prefer `dupeZ` for C interop strings** — it produces null-terminated slices.

### Common Pitfall: Forgetting to Free

```zig
// BAD: leak if append fails
const data = try allocator.alloc(u8, 100);
// ... no defer, no free

// GOOD: always pair with defer
const data = try allocator.alloc(u8, 100);
defer allocator.free(data);
```

### Common Pitfall: Wrong Allocator Lifetime

```zig
// BAD: arena is deinit'd before data is used
const data = arena_alloc.alloc(u8, 100) catch unreachable;
arena.deinit();
// data is now invalid!

// GOOD: data lifetime is within arena scope
var arena = std.heap.ArenaAllocator.init(backing);
defer arena.deinit();
const data = try arena.allocator().alloc(u8, 100);
// use data here, then arena.deinit() cleans up
```

---

## Chapter 10: Project — A Memory Pool Allocator

This project implements a fixed-size memory pool allocator. It pre-allocates a block of memory divided into equal-sized slots, and manages a free list for O(1) allocation and deallocation.

### src/main.zig

```zig
const std = @import("std");

const PoolAllocator = struct {
    buffer: []align(@alignOf(usize)) u8,
    slot_size: usize,
    free_list: usize, // index of first free slot, or std.math.maxInt(usize) if full
    slot_count: usize,

    const SLOT_FREE = std.math.maxInt(usize);

    fn init(buffer: []align(@alignOf(usize)) u8, slot_size: usize) PoolAllocator {
        const aligned_size = std.mem.alignForward(usize, slot_size, @alignOf(usize));
        const slot_count = buffer.len / aligned_size;
        // Build free list: each free slot stores the index of the next free slot
        for (0..slot_count) |i| {
            const offset = i * aligned_size;
            const next = if (i + 1 < slot_count) i + 1 else SLOT_FREE;
            @as(*usize, @ptrCast(@alignCast(buffer.ptr + offset))).* = next;
        }
        return .{
            .buffer = buffer,
            .slot_size = aligned_size,
            .free_list = if (slot_count > 0) 0 else SLOT_FREE,
            .slot_count = slot_count,
        };
    }

    fn allocator(self: *PoolAllocator) std.mem.Allocator {
        return .{
            .ptr = self,
            .vtable = &.{
                .alloc = allocFn,
                .resize = resizeFn,
                .free = freeFn,
            },
        };
    }

    fn allocFn(ptr: *anyopaque, len: usize, ptr_align: u8, ret_addr: usize) ?[*]u8 {
        _ = ret_addr;
        const self: *PoolAllocator = @ptrCast(@alignCast(ptr));
        _ = ptr_align; // pool provides alignment of @alignOf(usize)

        if (len > self.slot_size or self.free_list == SLOT_FREE) {
            return null; // Request too large or pool exhausted
        }

        const slot_idx = self.free_list;
        const slot_ptr = self.buffer.ptr + slot_idx * self.slot_size;
        // Read next free from the slot's storage
        self.free_list = @as(*usize, @ptrCast(@alignCast(slot_ptr))).*;
        return @ptrCast(slot_ptr);
    }

    fn resizeFn(ptr: *anyopaque, buf: []u8, buf_align: u8, new_len: usize, ret_addr: usize) bool {
        _ = ptr_align;
        _ = ret_addr;
        const self: *PoolAllocator = @ptrCast(@alignCast(ptr));
        if (new_len <= self.slot_size and buf.len <= self.slot_size) {
            // Already fits in the slot, no resize needed
            _ = self;
            _ = buf;
            _ = new_len;
            return true;
        }
        return false;
    }

    fn freeFn(ptr: *anyopaque, buf: []u8, buf_align: u8, ret_addr: usize) void {
        _ = buf_align;
        _ = ret_addr;
        const self: *PoolAllocator = @ptrCast(@alignCast(ptr));
        const slot_ptr = buf.ptr;
        // Push this slot back onto the free list
        @as(*usize, @ptrCast(@alignCast(slot_ptr))).* = self.free_list;
        self.free_list = @intCast((slot_ptr - self.buffer.ptr) / self.slot_size);
    }
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut().writer();

    // Allocate 16KB pool with 64-byte slots (256 slots)
    const pool_size = 16 * 1024;
    const slot_size: usize = 64;
    const pool_buffer = try init.allocator.alloc(u8, pool_size);
    defer init.allocator.free(pool_buffer);

    var pool = PoolAllocator.init(pool_buffer, slot_size);
    const pool_alloc = pool.allocator();

    try stdout.print("=== Memory Pool Allocator Demo ===\n", .{});
    try stdout.print("Pool: {} bytes, {} slots of {} bytes\n", .{
        pool_size, pool.slot_count, pool.slot_size,
    });

    // Allocate some objects
    var allocated: [10][]u8 = undefined;
    for (0..10) |i| {
        allocated[i] = try pool_alloc.alloc(u8, slot_size);
        // Fill with a pattern
        @memset(allocated[i], @intCast(i));
        try stdout.print("  Allocated slot {} at {*}\n", .{i, allocated[i].ptr});
    }

    try stdout.print("\nFreeing every other slot...\n", .{});
    for (0..10) |i| {
        if (i % 2 == 0) {
            pool_alloc.free(allocated[i]);
            try stdout.print("  Freed slot {}\n", .{i});
        }
    }

    try stdout.print("\nRe-allocating 3 slots...\n", .{});
    for (0..3) |i| {
        const new_slot = try pool_alloc.alloc(u8, slot_size);
        @memset(new_slot, 0xAA);
        try stdout.print("  Re-allocated slot at {*}\n", .{new_slot.ptr});
    }

    try stdout.print("\nPool allocator demo complete!\n", .{});
}
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "pool-allocator",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());

    const run_step = b.step("run", "Run the pool allocator demo");
    run_step.dependOn(&run_cmd.step);
}
```

### Summary

This book covered Zig 0.16's allocator model comprehensively. The key takeaway is that **explicit allocators give you control without complexity**. With ArenaAllocator now being lock-free and thread-safe, and ThreadSafeAllocator removed, the API is cleaner than ever. Use GPA for development, ArenaAllocator for batch allocation, FixedBufferAllocator for stack-only contexts, and build custom allocators for specialized needs.