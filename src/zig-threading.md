# Threading & Concurrency in Zig 0.16

A comprehensive guide to multithreaded programming in Zig, covering the renamed I/O-based synchronization primitives and the full threading API.

---

## Chapter 1: Introduction to Zig Threading

Zig provides first-class support for OS-level threads through `std.Thread`. The threading model is explicit: you create threads, pass data to them, and synchronize explicitly using mutexes, events, and atomic operations. There are no hidden runtime threads or implicit async tasks — every thread is one you asked for.

Zig 0.16 introduces an important naming reorganization. Several synchronization primitives that previously lived under `std.Thread` have been moved to `std.Io`:

| Old Name (pre-0.16) | New Name (0.16) |
|---------------------|-----------------|
| `std.Thread.Mutex` | `std.Io.Mutex` |
| `std.Thread.ResetEvent` | `std.Io.Event` |
| `std.Thread.WaitGroup` | `std.Io.Group` |
| `std.Thread.Futex` | `std.Io.Futex` |

However, `std.Thread.spawn` and `std.Thread.Pool` remain unchanged in location.

The rationale for this move is that these primitives are I/O-oriented synchronization tools that integrate with Zig's I/O subsystem, not just threading tools. The `std.Io` namespace reflects their role in coordinating I/O operations across threads.

---

## Chapter 2: std.Thread.spawn — Creating Threads

The fundamental way to create a thread in Zig is `std.Thread.spawn`. It takes a configuration struct and a function pointer with its arguments:

```zig
const std = @import("std");

fn worker(id: u32) void {
    std.log.info("Thread {} starting", .{id});
    std.time.sleep(1 * std.time.ns_per_s);
    std.log.info("Thread {} done", .{id});
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    const thread = try std.Thread.spawn(.{}, worker, .{@as(u32, 42)});
    thread.join();
    std.log.info("Main thread done", .{});
}
```

### Thread Configuration

The first argument to `std.Thread.spawn` is a configuration struct. Common options include:

```zig
const thread = try std.Thread.spawn(.{
    .stack_size = 8 * 1024 * 1024, // 8 MB stack
}, worker, .{@as(u32, 1)});
```

By default, Zig uses the OS default stack size (typically 8 MB on Linux, 1 MB on Windows).

### Multiple Threads

```zig
const std = @import("std");

fn worker(id: usize) void {
    std.log.info("Worker {} running on thread {}", .{ id, std.Thread.getCurrentId() });
}

pub fn main(init: std.process.Init) !void {
    _ = init;
    const num_threads = 4;

    var threads: [num_threads]std.Thread = undefined;
    for (&threads, 0..) |*t, i| {
        t.* = try std.Thread.spawn(.{}, worker, .{i});
    }
    for (&threads) |t| {
        t.join();
    }
    std.log.info("All workers complete", .{});
}
```

---

## Chapter 3: Thread Data — Passing Arguments and Return Values

### Passing Multiple Arguments

Zig's `std.Thread.spawn` passes arguments as a tuple. This is a significant advantage over C's `void*` pointer:

```zig
const std = @import("std");

const WorkItem = struct {
    start: usize,
    end: usize,
    name: []const u8,
};

fn processRange(item: WorkItem) void {
    std.log.info("{s}: processing range {}..{}", .{ item.name, item.start, item.end });
    var sum: usize = 0;
    for (item.start..item.end) |i| {
        sum += i;
    }
    std.log.info("{s}: sum = {}", .{ item.name, sum });
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    const thread = try std.Thread.spawn(.{}, processRange, .{
        .{
            .start = 1,
            .end = 100,
            .name = "chunk-A",
        },
    });
    thread.join();
}
```

### Returning Values via Shared Pointers

Since thread functions do not return values to the caller (they return `void`), use shared pointers with proper synchronization:

```zig
const std = @import("std");

fn compute(ptr: *usize) void {
    var sum: usize = 0;
    for (0..1_000_000) |i| {
        sum += i;
    }
    ptr.* = sum;
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var result: usize = 0;
    const thread = try std.Thread.spawn(.{}, compute, .{&result});
    thread.join();

    const writer = std.io.getStdOut().writer();
    try writer.print("Result: {}\n", .{result});
}
```

> **Important**: The above is safe only because `thread.join()` provides a happens-before guarantee. The parent thread reads `result` only after the child thread has completed.

---

## Chapter 4: Thread Detachment and Joining

### Joining (Default Behavior)

By default, spawned threads must be joined. If a thread is not joined, the runtime may report a resource leak:

```zig
const thread = try std.Thread.spawn(.{}, worker, .{});
// Must call thread.join() before the thread handle goes out of scope
thread.join();
```

### Detached Threads

You can detach a thread so it runs independently. The thread's resources are automatically reclaimed when it finishes:

```zig
const std = @import("std");

fn backgroundWorker() void {
    std.time.sleep(2 * std.time.ns_per_s);
    std.log.info("Background work complete", .{});
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    // Detach — don't need to join
    const thread = try std.Thread.spawn(.{}, backgroundWorker, .{});
    thread.detach();

    std.log.info("Main thread exiting (background continues)", .{});
}
```

### The Join Pattern

The most common pattern is to spawn multiple worker threads and then join them all:

```zig
const std = @import("std");

fn worker(id: usize, counter: *std.atomic.Value(usize)) void {
    _ = counter.fetchAdd(1, .seq_cst);
    std.log.info("Worker {} finished", .{id});
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var counter = std.atomic.Value(usize).init(0);
    var threads: [8]std.Thread = undefined;

    for (&threads, 0..) |*t, i| {
        t.* = try std.Thread.spawn(.{}, worker, .{ i, &counter });
    }

    for (&threads) |t| {
        t.join();
    }

    const writer = std.io.getStdOut().writer();
    try writer.print("All {} workers done. Counter: {}\n", .{ threads.len, counter.load(.seq_cst) });
}
```

---

## Chapter 5: std.Io.Mutex (was std.Thread.Mutex)

Zig 0.16 moves `Mutex` from `std.Thread` to `std.Io`. The API remains the same, but the import path changes:

```zig
const std = @import("std");

// In Zig 0.16: use std.Io.Mutex
// In pre-0.16: use std.Thread.Mutex

var counter_mutex: std.Io.Mutex = .{};
var counter: usize = 0;

fn incrementMany(n: usize) void {
    for (0..n) |_| {
        counter_mutex.lock();
        defer counter_mutex.unlock();
        counter += 1;
    }
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threads: [4]std.Thread = undefined;
    for (&threads) |*t| {
        t.* = try std.Thread.spawn(.{}, incrementMany, .{@as(usize, 100_000)});
    }
    for (&threads) |t| {
        t.join();
    }

    const writer = std.io.getStdOut().writer();
    try writer.print("Counter (with mutex): {}\n", .{counter});
}
```

### TryLock

`std.Io.Mutex` also provides `tryLock`, which returns `false` instead of blocking if the mutex is already held:

```zig
if (mutex.tryLock()) {
    defer mutex.unlock();
    // critical section
} else {
    // mutex is busy, do something else
}
```

### RAII with defer

Always use `defer mutex.unlock()` immediately after `mutex.lock()` to ensure the mutex is released even if the critical section panics:

```zig
fn safeUpdate(shared: *SharedData) void {
    shared.mutex.lock();
    defer shared.mutex.unlock();

    // If any of these operations panic, the mutex is still released
    shared.value += 1;
    shared.timestamp = std.time.nanoTimestamp();
}
```

---

## Chapter 6: std.Io.Event (was std.Thread.ResetEvent)

`std.Io.Event` (previously `std.Thread.ResetEvent`) is a manual-reset event that allows one thread to signal another:

```zig
const std = @import("std");

var event: std.Io.Event = .{};

fn waiter(id: usize) void {
    std.log.info("Waiter {} waiting for signal...", .{id});
    event.wait();
    std.log.info("Waiter {} received signal!", .{id});
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threads: [3]std.Thread = undefined;
    for (&threads, 0..) |*t, i| {
        t.* = try std.Thread.spawn(.{}, waiter, .{i});
    }

    std.time.sleep(1 * std.time.ns_per_s);
    std.log.info("Main: signaling all waiters", .{});
    event.set(); // Wakes ALL waiting threads

    for (&threads) |t| t.join();
}
```

### Auto-Reset vs Manual-Reset

`std.Io.Event` is a manual-reset event. Once `set()` is called, all current and future `wait()` calls return immediately until `reset()` is called:

```zig
event.set();
event.wait(); // returns immediately
event.wait(); // returns immediately
event.reset();
event.wait(); // blocks until next set()
```

### Event for One-Shot Initialization

```zig
var initialized: std.Io.Event = .{};
var config_value: usize = 0;

fn lazyInit() void {
    if (config_value == 0) {
        config_value = 42;
        initialized.set();
    }
}

fn user(id: usize) void {
    initialized.wait();
    std.log.info("User {} sees config={}", .{ id, config_value });
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threads: [3]std.Thread = undefined;
    for (&threads, 0..) |*t, i| {
        t.* = try std.Thread.spawn(.{}, user, .{i});
    }

    std.time.sleep(500 * std.time.ns_per_ms);
    lazyInit();

    for (&threads) |t| t.join();
}
```

---

## Chapter 7: std.Io.Group (was std.Thread.WaitGroup)

`std.Io.Group` (previously `std.Thread.WaitGroup`) allows you to wait for multiple threads or operations to complete:

```zig
const std = @import("std");

fn worker(id: usize, wg: *std.Io.Group) void {
    defer wg.finish();
    std.log.info("Worker {} starting", .{id});
    std.time.sleep(@as(u64, @intCast(id)) * 100 * std.time.ns_per_ms);
    std.log.info("Worker {} done", .{id});
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var wg: std.Io.Group = .{};
    defer wg.wait();

    var threads: [5]std.Thread = undefined;
    for (&threads, 0..) |*t, i| {
        wg.start();
        t.* = try std.Thread.spawn(.{}, worker, .{ i, &wg });
    }

    for (&threads) |t| t.join();

    // wg.wait() is called by defer after all threads are joined
    std.log.info("All workers reported complete", .{});
}
```

### Group Pattern Explanation

1. `wg.start()` — Increments an internal counter
2. `wg.finish()` — Decrements the counter (called via `defer` in each worker)
3. `wg.wait()` — Blocks until the counter reaches zero

This is particularly useful when you don't want to keep track of individual thread handles, or when tasks are dispatched dynamically:

```zig
const std = @import("std");

fn processFile(path: []const u8, wg: *std.Io.Group) void {
    defer wg.finish();
    std.log.info("Processing: {s}", .{path});
    // Simulate work
    std.time.sleep(100 * std.time.ns_per_ms);
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    const files = [_][]const u8{
        "file1.txt", "file2.txt", "file3.txt",
        "file4.txt", "file5.txt",
    };

    var wg: std.Io.Group = .{};
    defer wg.wait();

    var threads: [files.len]std.Thread = undefined;
    for (&files, 0..) |f, i| {
        wg.start();
        threads[i] = try std.Thread.spawn(.{}, processFile, .{ f, &wg });
    }

    for (&threads) |t| t.join();

    std.log.info("All files processed", .{});
}
```

---

## Chapter 8: std.Io.Futex (was std.Thread.Futex)

`std.Io.Futex` (previously `std.Thread.Futex`) provides low-level synchronization primitives based on the futex system call. Most code should use `Mutex`, `Event`, or `Group` instead, but `Futex` is useful for building custom synchronization constructs:

```zig
const std = @import("std");

var futex_val: std.Io.Futex.Value = .{};

fn waiterFn(id: usize) void {
    std.log.info("Waiter {}: about to wait", .{id});
    std.Io.Futex.wait(&futex_val, 0);
    std.log.info("Waiter {}: woke up!", .{id});
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threads: [3]std.Thread = undefined;
    for (&threads, 0..) |*t, i| {
        t.* = try std.Thread.spawn(.{}, waiterFn, .{i});
    }

    std.time.sleep(1 * std.time.ns_per_s);
    std.log.info("Main: waking all waiters", .{});
    std.Io.Futex.wake(&futex_val, std.Io.Futex.WakeScope.all);

    for (&threads) |t| t.join();
}
```

### Building a Simple Spinlock with Futex

```zig
const std = @import("std");

const SimpleLock = struct {
    state: std.atomic.Value(u32) = std.atomic.Value(u32).init(0),

    fn acquire(self: *SimpleLock) void {
        while (self.state.cmpxchgWeak(
            0,
            1,
            .acquire,
            .monotonic,
        )) |result| {
            if (result == null) return; // success
            // Wait for the state to change
            std.Io.Futex.wait(&self.state, 1);
        }
    }

    fn release(self: *SimpleLock) void {
        self.state.store(0, .release);
        std.Io.Futex.wake(&self.state, 1);
    }
};

var lock: SimpleLock = .{};
var protected_value: usize = 0;

fn increment(n: usize) void {
    for (0..n) |_| {
        lock.acquire();
        defer lock.release();
        protected_value += 1;
    }
}

pub fn main(init: std.process.Init) !void {
    _ = init;

    var threads: [4]std.Thread = undefined;
    for (&threads) |*t| {
        t.* = try std.Thread.spawn(.{}, increment, .{@as(usize, 50_000)});
    }
    for (&threads) |t| t.join();

    const writer = std.io.getStdOut().writer();
    try writer.print("Protected value (futex lock): {}\n", .{protected_value});
}
```

---

## Chapter 9: std.Thread.Pool — Thread Pools

`std.Thread.Pool` (unchanged in 0.16) provides a fixed-size thread pool for executing work items:

```zig
const std = @import("std");

fn workerTask(ctx: *const struct {
    id: usize,
    value: usize,
}) void {
    std.log.info("Task {}: computing sum up to {}", .{ ctx.id, ctx.value });
    var sum: usize = 0;
    for (0..ctx.value) |i| sum += i;
    std.log.info("Task {}: result = {}", .{ ctx.id, sum });
}

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var pool: std.Thread.Pool = undefined;
    try pool.init(.{
        .allocator = allocator,
        .n_threads = 4,
    });
    defer pool.deinit();

    const writer = std.io.getStdOut().writer();
    try writer.writeAll("Dispatching 10 tasks to thread pool...\n");

    var tasks: [10]struct { id: usize, value: usize } = undefined;
    for (&tasks, 0..) |*t, i| {
        t.id = i;
        t.value = (i + 1) * 10_000;
        pool.spawn(workerTask, .{t});
    }

    pool.wait();
    try writer.writeAll("All tasks complete!\n");
}
```

### Thread Pool with closures

For more complex work, you can use `pool.spawnW` which allows passing extra context and cleanup functions:

```zig
const std = @import("std");

fn processItem(item: *const u32) void {
    std.log.info("Processing item: {}", .{item.*});
    std.time.sleep(10 * std.time.ns_per_ms);
}

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var pool: std.Thread.Pool = undefined;
    try pool.init(.{
        .allocator = allocator,
        .n_threads = 3,
    });
    defer pool.deinit();

    var items: [20]u32 = undefined;
    for (&items, 0..) |*item, i| {
        item.* = @intCast(i);
    }

    for (&items) |*item| {
        pool.spawn(processItem, .{item});
    }

    pool.wait();
    std.log.info("All items processed", .{});
}
```

---

## Chapter 10: Atomic Operations — std.atomic

Zig's `std.atomic` module provides lock-free atomic operations. In 0.16, atomics use the `std.atomic.Value(T)` wrapper:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    var counter = std.atomic.Value(u64).init(0);

    // Atomic load
    const val = counter.load(.seq_cst);
    _ = val;

    // Atomic store
    counter.store(42, .seq_cst);

    // Atomic fetch-and-add
    const old = counter.fetchAdd(1, .seq_cst);
    std.log.info("Old value: {}, new value: {}", .{ old, counter.load(.seq_cst) });

    // Compare-and-swap
    const result = counter.cmpxchgStrong(43, 100, .seq_cst, .seq_cst);
    if (result != null) {
        std.log.info("CAS succeeded, old value was {}", .{result.?});
    } else {
        std.log.info("CAS failed, current value is {}", .{counter.load(.seq_cst)});
    }
}
```

### Memory Ordering

Zig exposes the full C++ memory model through `std.atomic.Ordering`:

| Ordering | Description |
|----------|-------------|
| `.relaxed` | No ordering guarantees |
| `.acquire` | All reads before this are not reordered past it |
| `.release` | All writes after this are not reordered before it |
| `.acq_rel` | Both acquire and release |
| `.seq_cst` | Sequentially consistent (strongest) |

### Lock-Free Queue with Atomics

```zig
const std = @import("std");

const Node = struct {
    next: ?*Node = null,
    value: u32,
};

const LockFreeStack = struct {
    head: std.atomic.Value(?*Node) = std.atomic.Value(?*Node).init(null),

    fn push(self: *LockFreeStack, node: *Node) void {
        node.next = self.head.load(.acquire);
        while (self.head.cmpxchgWeak(
            node.next,
            node,
            .release,
            .acquire,
        )) |result| {
            node.next = result.?;
        }
    }

    fn pop(self: *LockFreeStack) ?*Node {
        while (true) {
            const head = self.head.load(.acquire) orelse return null;
            const next = head.next;
            if (self.head.cmpxchgStrong(head, next, .acq_rel, .acquire)) |_| {
                return head;
            }
        }
    }
};

pub fn main(init: std.process.Init) !void {
    _ = init;

    var stack: LockFreeStack = .{};
    var nodes: [10]Node = undefined;
    for (&nodes, 0..) |*n, i| {
        n.value = @intCast(i);
        stack.push(n);
    }

    const writer = std.io.getStdOut().writer();
    while (stack.pop()) |node| {
        try writer.print("Popped: {}\n", .{node.value});
    }
}
```

---

## Chapter 11: Project — A Parallel File Search / Word Counter

This project implements a multi-threaded file scanner that searches for files in a directory and counts occurrences of a target word across all files, using thread pools and atomic counters.

### `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "parallel-search",
        .root_source_file = b.path("src/main.zig"),
        .target = target,
        .optimize = optimize,
    });
    b.installArtifact(exe);
}
```

### `src/main.zig`

```zig
const std = @import("std");

const SearchState = struct {
    target_word: []const u8,
    total_matches: std.atomic.Value(usize) = std.atomic.Value(usize).init(0),
    files_scanned: std.atomic.Value(usize) = std.atomic.Value(usize).init(0),
    total_bytes: std.atomic.Value(usize) = std.atomic.Value(usize).init(0),
};

fn countWordInBytes(data: []const u8, target: []const u8) usize {
    var count: usize = 0;
    var i: usize = 0;
    while (i < data.len) {
        if (i + target.len <= data.len and
            std.mem.eql(u8, data[i..][0..target.len], target))
        {
            count += 1;
            i += target.len;
        } else {
            i += 1;
        }
    }
    return count;
}

fn searchFile(path: []const u8, state: *SearchState) void {
    const allocator = std.heap.page_allocator;

    const file = std.fs.cwd().openFile(path, .{}) catch |err| {
        std.log.warn("Cannot open {s}: {}", .{ path, err });
        return;
    };
    defer file.close();

    const stat = file.stat() catch return;
    if (stat.size > 10 * 1024 * 1024) return; // Skip files > 10 MB

    const data = file.readToEndAlloc(allocator, stat.size) catch |err| {
        std.log.warn("Cannot read {s}: {}", .{ path, err });
        return;
    };
    defer allocator.free(data);

    const matches = countWordInBytes(data, state.target_word);

    _ = state.total_matches.fetchAdd(matches, .relaxed);
    _ = state.files_scanned.fetchAdd(1, .relaxed);
    _ = state.total_bytes.fetchAdd(data.len, .relaxed);

    if (matches > 0) {
        std.log.info("{s}: {} matches", .{ path, matches });
    }
}

fn scanDir(dir_path: []const u8, pool: *std.Thread.Pool, state: *SearchState, allocator: std.mem.Allocator) !void {
    var dir = try std.fs.cwd().openDir(dir_path, .{ .iterate = true });
    defer dir.close();

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        const full_path = try std.fs.path.join(allocator, &.{ dir_path, entry.name });
        defer allocator.free(full_path);

        if (entry.kind == .directory) {
            try scanDir(full_path, pool, state, allocator);
        } else if (entry.kind == .file) {
            const owned = try allocator.dupeZ(u8, full_path);
            pool.spawn(searchFile, .{ owned, state });
        }
    }
}

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    const args = init.minimal.args;
    if (args.len < 3) {
        const writer = std.io.getStdErr().writer();
        try writer.writeAll("Usage: parallel-search <directory> <word>\n");
        std.process.exit(1);
    }

    const dir_path = args[1];
    const target_word = args[2];

    var state = SearchState{ .target_word = target_word };

    var pool: std.Thread.Pool = undefined;
    try pool.init(.{
        .allocator = allocator,
        .n_threads = @min(std.Thread.getCpuCount() - 1, 8),
    });
    defer pool.deinit();

    const writer = std.io.getStdOut().writer();
    try writer.print("Searching for '{s}' in {s}...\n", .{ target_word, dir_path });

    const start = std.time.nanoTimestamp();
    try scanDir(dir_path, &pool, &state, allocator);
    pool.wait();
    const elapsed = @as(f64, @floatFromInt(std.time.nanoTimestamp() - start)) / 1_000_000.0;

    try writer.print("\n=== Results ===\n", .{});
    try writer.print("Files scanned:  {}\n", .{state.files_scanned.load(.seq_cst)});
    try writer.print("Total matches:  {}\n", .{state.total_matches.load(.seq_cst)});
    try writer.print("Bytes scanned:  {}\n", .{state.total_bytes.load(.seq_cst)});
    try writer.print("Time:           {d:.2} ms\n", .{elapsed});
}
```

### Running the Project

```bash
zig build run -- . "the"
```

Expected output:

```
Searching for 'the' in . ...
src/main.zig: 3 matches
README.md: 12 matches
src/utils.zig: 1 match

=== Results ===
Files scanned:  3
Total matches:  16
Bytes scanned:  8472
Time:           4.23 ms
```

---

## Summary

Zig 0.16's threading and concurrency story is explicit and powerful:

1. **`std.Thread.spawn`** remains the primary way to create OS threads
2. **`std.Io.Mutex`** (moved from `std.Thread.Mutex`) provides mutual exclusion
3. **`std.Io.Event`** (moved from `std.Thread.ResetEvent`) enables signaling between threads
4. **`std.Io.Group`** (moved from `std.Thread.WaitGroup`) coordinates multiple threads
5. **`std.Io.Futex`** (moved from `std.Thread.Futex`) offers low-level synchronization
6. **`std.Thread.Pool`** provides managed thread pools for work distribution
7. **`std.atomic.Value(T)`** enables lock-free programming with full memory ordering control

The move of synchronization primitives from `std.Thread` to `std.Io` reflects Zig's growing emphasis on an I/O-centric architecture where threading and I/O are first-class partners, not separate concerns.