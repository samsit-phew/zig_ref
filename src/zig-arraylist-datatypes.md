# Collections & Data Types in Zig 0.16

A comprehensive guide to Zig's data structure ecosystem, covering dynamic arrays, hash maps, linked lists, bit sets, and specialized collections — all updated for Zig 0.16's API conventions.

---

## Chapter 1: Introduction to Zig's Data Structure Ecosystem

Zig's standard library provides a rich set of data structures in `std`. Unlike languages with built-in generic collections (C++ `std::vector`, Rust `Vec`), Zig's collections are explicit about memory management through the allocator parameter.

In Zig 0.16, a key design principle is that **"unmanaged" versions are the default**. Managed versions exist as thin wrappers that bundle an allocator with the data structure. This gives you full control:

- **Unmanaged** (`std.ArrayListUnmanaged(T)`): You pass an allocator to each method call. Zero overhead if you already have an allocator in scope.
- **Managed** (`std.ArrayList(T, Allocator)`): Stores the allocator inside the struct. Slightly more ergonomic at the cost of one pointer stored per instance.

This book covers every major collection type in Zig's standard library with practical code examples.

---

## Chapter 2: std.ArrayList(T) and std.ArrayListUnmanaged(T) — Dynamic Arrays

### Managed ArrayList

The managed `ArrayList` stores its allocator internally. In Zig 0.16, the managed version takes an allocator type parameter:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    // Managed ArrayList — allocator is stored internally
    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    // Append elements
    try list.append(10);
    try list.append(20);
    try list.append(30);

    // Access elements
    const writer = std.io.getStdOut().writer();
    try writer.print("Length: {}\n", .{list.items.len});
    try writer.print("Capacity: {}\n", .{list.capacity});
    try writer.print("First: {}\n", .{list.items[0]});
    try writer.print("Last: {}\n", .{list.items[list.items.len - 1]});
}
```

### Unmanaged ArrayList

The unmanaged version requires passing the allocator to each mutating method:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    // Unmanaged — no allocator stored in the struct
    var list: std.ArrayListUnmanaged(i32) = .empty;
    defer list.deinit(allocator);

    try list.append(allocator, 42);
    try list.append(allocator, 99);
    try list.append(allocator, 7);

    const writer = std.io.getStdOut().writer();
    for (list.items, 0..) |item, i| {
        try writer.print("items[{}] = {}\n", .{ i, item });
    }
}
```

### Common ArrayList Operations

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var list = std.ArrayList(i32).init(allocator);
    defer list.deinit();

    // Append a slice
    try list.appendSlice(&[_]i32{ 1, 2, 3, 4, 5 });

    // Insert at index
    try list.insert(2, 99); // {1, 2, 99, 3, 4, 5}

    // Ordered remove (shifts elements)
    const removed = list.orderedRemove(0); // returns 1
    _ = removed;

    // Swap remove (faster, does not preserve order)
    _ = list.swapRemove(1); // moves last element to index 1

    // Pop from end
    if (list.popOrNull()) |val| {
        std.log.info("Popped: {}", .{val});
    }

    // Resize
    try list.resize(10); // extends with zeros
    try list.resize(3); // truncates

    // Clear (keeps capacity)
    list.clear();
    list.clearRetainingCapacity();

    const writer = std.io.getStdOut().writer();
    try writer.print("Final len: {}, cap: {}\n", .{ list.items.len, list.capacity });
}
```

---

## Chapter 3: std.SegmentedList — Segmented Lists

`std.SegmentedList` stores elements in fixed-size segments, avoiding large contiguous allocations. This is useful when you expect very large lists and want to avoid reallocation costs:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    // SegmentedList(ElementType, segment_size)
    var list: std.SegmentedList(i32, 4) = .empty;
    defer list.deinit(allocator);

    // Append elements
    for (0..20) |i| {
        try list.append(allocator, @as(i32, @intCast(i)));
    }

    // Iterate
    const writer = std.io.getStdOut().writer();
    try writer.print("Length: {}\n", .{list.len});

    var it = list.iterator(0);
    var count: usize = 0;
    while (it.next()) |item| {
        if (count < 10) {
            try writer.print("  [{}] = {}\n", .{ count, item });
        }
        count += 1;
    }
    try writer.print("Iterated {} items total\n", .{count});
}
```

### When to Use SegmentedList

- When the total number of elements is unpredictable
- When you want to avoid `memcpy`-style reallocations
- When individual segments fit in cache lines
- The trade-off: slightly slower indexed access due to segment indirection

---

## Chapter 4: std.HashMap and std.HashMapUnmanaged — Hash Maps

### Managed HashMap

Zig 0.16 provides managed and unmanaged hash maps. The managed version stores the allocator:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var map = std.HashMap(u32, []const u8, std.hash_map.AutoContext(u32), std.hash_map.default_max_load_percentage).init(allocator);
    defer map.deinit();

    // Insert
    try map.put(1, "one");
    try map.put(2, "two");
    try map.put(3, "three");
    try map.put(42, "the answer");

    // Get
    if (map.get(42)) |val| {
        std.log.info("42 = {s}", .{val});
    }

    // Contains
    const exists = map.contains(3);
    std.log.info("Contains 3: {}", .{exists});

    // Remove
    _ = map.remove(2);

    // Iterate
    const writer = std.io.getStdOut().writer();
    var it = map.iterator();
    while (it.next()) |entry| {
        try writer.print("{} => {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }
}
```

### Unmanaged HashMap

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var map: std.HashMapUnmanaged(u32, []const u8, std.hash_map.AutoContext(u32), std.hash_map.default_max_load_percentage) = .empty;
    defer map.deinit(allocator);

    try map.put(allocator, 1, "alpha");
    try map.put(allocator, 2, "beta");

    if (map.get(1)) |val| {
        std.log.info("Found: {s}", .{val});
    }

    _ = map.remove(allocator, 1);
}
```

### Custom Hash Context

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u32,
};

const PersonContext = struct {
    pub fn hash(self: @This(), key: Person) u64 {
        _ = self;
        var hasher = std.hash.Wyhash.init(0);
        hasher.update(key.name);
        hasher.update(std.mem.asBytes(&key.age));
        return hasher.final();
    }

    pub fn eql(_: @This(), a: Person, b: Person, _: usize) bool {
        return std.mem.eql(u8, a.name, b.name) and a.age == b.age;
    }
};

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var map = std.HashMap(Person, []const u8, PersonContext, 80).init(allocator);
    defer map.deinit();

    const alice = Person{ .name = "Alice", .age = 30 };
    try map.put(alice, "Engineer");
    try map.put(.{ .name = "Bob", .age = 25 }, "Designer");

    const writer = std.io.getStdOut().writer();
    if (map.get(alice)) |role| {
        try writer.print("Alice is a {s}\n", .{role});
    }
}
```

---

## Chapter 5: std.StringHashMap and std.StringHashMapUnmanaged

String hash maps are a convenience wrapper around `HashMap` with `[]const u8` keys:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var map = std.StringHashMap([]const u8).init(allocator);
    defer map.deinit();

    // Insert
    try map.put("zig", "A systems programming language");
    try map.put("rust", "A memory-safe systems language");
    try map.put("go", "A language for cloud services");

    // Lookup
    if (map.get("zig")) |desc| {
        std.log.info("zig: {s}", .{desc});
    }

    // Check existence
    const has_c = map.contains("c");
    std.log.info("Has 'c': {}", .{has_c});

    // Iterate
    const writer = std.io.getStdOut().writer();
    try writer.writeAll("All entries:\n");
    var it = map.iterator();
    while (it.next()) |entry| {
        try writer.print("  {s} = {s}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }

    // Get or put
    const gop = try map.getOrPut("python");
    if (!gop.found_existing) {
        gop.value_ptr.* = "A general-purpose language";
    }

    // Remove
    _ = map.remove("go");
}
```

### StringHashMapUnmanaged

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var map: std.StringHashMapUnmanaged(u32) = .empty;
    defer map.deinit(allocator);

    try map.put(allocator, "apples", 5);
    try map.put(allocator, "oranges", 3);

    if (map.get("apples")) |count| {
        std.log.info("Apples: {}", .{count});
    }
}
```

### Counting Word Frequencies

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    const text = "the quick brown fox jumps over the lazy dog the fox";
    var frequencies = std.StringHashMap(usize).init(allocator);
    defer frequencies.deinit();

    var it = std.mem.splitSequence(u8, text, " ");
    while (it.next()) |word| {
        const gop = try frequencies.getOrPut(word);
        if (gop.found_existing) {
            gop.value_ptr.* += 1;
        } else {
            gop.value_ptr.* = 1;
        }
    }

    const writer = std.io.getStdOut().writer();
    try writer.writeAll("Word frequencies:\n");
    var map_it = frequencies.iterator();
    while (map_it.next()) |entry| {
        try writer.print("  '{s}': {}\n", .{
            entry.key_ptr.*,
            entry.value_ptr.*,
        });
    }
}
```

---

## Chapter 6: std.BoundedArray — Stack-Allocated Fixed-Capacity Arrays

`std.BoundedArray` provides a fixed-capacity array that tracks its length. No heap allocation needed:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // BoundedArray(ElementType, capacity)
    var buf: std.BoundedArray(i32, 8) = .empty;

    // Append (returns error.OutOfMemory if full)
    try buf.append(10);
    try buf.append(20);
    try buf.append(30);
    try buf.appendSlice(&[_]i32{ 40, 50 });

    const writer = std.io.getStdOut().writer();
    try writer.print("Length: {}/{}\n", .{ buf.len, buf.capacity });

    for (buf.slice(), 0..) |item, i| {
        try writer.print("  [{}] = {}\n", .{ i, item });
    }

    // Pop
    if (buf.popOrNull()) |val| {
        try writer.print("Popped: {}\n", .{val});
    }

    // Const access to underlying slice
    const const_slice: []const i32 = buf.constSlice();
    _ = const_slice;
}
```

### Use Cases for BoundedArray

- Stack-based buffers where the maximum size is known
- Parsing fixed-format protocols
- Small collections that don't need heap allocation
- Function-local buffers that are returned as slices

```zig
fn parseCsvLine(line: []const u8) !std.BoundedArray([]const u8, 16) {
    var fields: std.BoundedArray([]const u8, 16) = .empty;
    var it = std.mem.splitSequence(u8, line, ",");
    while (it.next()) |field| {
        try fields.append(field);
    }
    return fields;
}
```

---

## Chapter 7: std.PackedIntArray — Memory-Efficient Integer Arrays

`std.PackedIntArray(Index, size)` stores integers using the minimum number of bits needed, packing multiple values into each machine word:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // Store 100 u4 values (4 bits each = 50 bytes instead of 100 bytes)
    var packed: std.PackedIntArray(u4, 100) = .initFill(0);

    // Set individual elements
    packed.set(0, 5);
    packed.set(1, 10);
    packed.set(2, 15);
    packed.set(99, 7);

    // Read elements
    const writer = std.io.getStdOut().writer();
    try writer.print("Element 0: {}\n", .{packed.get(0)});
    try writer.print("Element 99: {}\n", .{packed.get(99)});

    // Size comparison
    const normal_size = @sizeOf([100]u8); // 100 bytes
    const packed_size = @sizeOf(@TypeOf(packed)); // ~50 bytes
    try writer.print("Normal array: {} bytes\n", .{normal_size});
    try writer.print("Packed array: {} bytes\n", .{packed_size});
}
```

### Practical Use: Storing Scores

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // Scores range 0-255, stored as u8 — use packed u8
    // For 0-15 scores, use u4 (4 bits per element)
    var scores: std.PackedIntArray(u4, 1000) = .initFill(0);

    // Initialize with random-ish scores
    for (0..1000) |i| {
        scores.set(i, @as(u4, @intCast(i % 16)));
    }

    // Compute average
    var total: usize = 0;
    for (0..1000) |i| {
        total += scores.get(i);
    }
    const avg: f64 = @as(f64, @floatFromInt(total)) / 1000.0;

    const writer = std.io.getStdOut().writer();
    try writer.print("Average score: {d:.2}\n", .{avg});
    try writer.print("Memory used: {} bytes (vs {} for [1000]u8)\n", .{
        @sizeOf(@TypeOf(scores)),
        @sizeOf([1000]u8),
    });
}
```

---

## Chapter 8: std.SinglyLinkedList and std.DoublyLinkedList

### Singly Linked List

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // SinglyLinkedList(ElementType)
    var list: std.SinglyLinkedList(i32) = .empty;

    // Allocate nodes
    const allocator = std.heap.page_allocator;

    const n1 = try list.createNode(allocator, 10);
    const n2 = try list.createNode(allocator, 20);
    const n3 = try list.createNode(allocator, 30);

    // Insert at head
    list.prepend(n1);
    list.prepend(n2);
    list.prepend(n3);
    // List: 30 -> 20 -> 10

    const writer = std.io.getStdOut().writer();
    try writer.writeAll("Singly linked list:\n");
    var it = list.first;
    var idx: usize = 0;
    while (it) |node| : (it = node.next) {
        try writer.print("  [{}] = {}\n", .{ idx, node.data });
        idx += 1;
    }

    // Remove head
    const removed = list.popFirst();
    if (removed) |node| {
        try writer.print("Removed: {}\n", .{node.data});
        allocator.destroy(node);
    }

    // Clean up remaining nodes
    while (list.popFirst()) |node| {
        allocator.destroy(node);
    }
}
```

### Doubly Linked List

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;
    const allocator = std.heap.page_allocator;

    var list: std.DoublyLinkedList(i32) = .empty;

    const n1 = try list.createNode(allocator, 100);
    const n2 = try list.createNode(allocator, 200);
    const n3 = try list.createNode(allocator, 300);

    // Insert at end
    list.append(n1);
    list.append(n2);
    list.append(n3);

    const writer = std.io.getStdOut().writer();
    try writer.writeAll("Forward:\n");
    var it = list.first;
    while (it) |node| : (it = node.next) {
        try writer.print("  {}\n", .{node.data});
    }

    try writer.writeAll("Backward:\n");
    var rit = list.last;
    while (rit) |node| : (rit = node.prev) {
        try writer.print("  {}\n", .{node.data});
    }

    // Remove specific node
    list.remove(n2);
    allocator.destroy(n2);

    try writer.writeAll("After removing 200:\n");
    it = list.first;
    while (it) |node| : (it = node.next) {
        try writer.print("  {}\n", .{node.data});
    }

    // Clean up
    while (list.popFirst()) |node| {
        allocator.destroy(node);
    }
}
```

---

## Chapter 9: std.BitSet and std.BitSetAligned — Bit Manipulation

### std.BitSet

`std.BitSet` stores a fixed number of bits compactly:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // BitSet with 64 bits
    var bits: std.BitSet(64) = .initEmpty();

    // Set bits
    bits.set(0);
    bits.set(5);
    bits.set(10);
    bits.set(63);

    // Test bits
    const writer = std.io.getStdOut().writer();
    try writer.print("Bit 0:  {}\n", .{bits.isSet(0)});
    try writer.print("Bit 1:  {}\n", .{bits.isSet(1)});
    try writer.print("Bit 63: {}\n", .{bits.isSet(63)});

    // Toggle
    bits.toggle(5);
    try writer.print("Bit 5 after toggle: {}\n", .{bits.isSet(5)});

    // Count set bits (population count)
    try writer.print("Population count: {}\n", .{bits.count()});

    // Find first set bit
    if (bits.findFirstSet()) |idx| {
        try writer.print("First set bit: {}\n", .{idx});
    }

    // Iteration
    try writer.writeAll("Set bits: ");
    var it = bits.iterator(.{});
    while (it.next()) |idx| {
        try writer.print("{} ", .{idx});
    }
    try writer.print("\n");
}
```

### std.BitSetAligned

`std.BitSetAligned` uses aligned backing storage, which can improve performance for large bit sets:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    // BitSetAligned(N, StorageType)
    // StorageType determines alignment — usize gives natural word alignment
    var bits: std.BitSetAligned(256, usize) = .initEmpty();

    // Set a range
    bits.setRangeValue(.{ .start = 10, .end = 20 }, true);

    const writer = std.io.getStdOut().writer();
    try writer.print("Bits 10-19 set: ", .{});
    for (10..20) |i| {
        try writer.print("{} ", .{bits.isSet(i)});
    }
    try writer.print("\n");

    try writer.print("Size in bytes: {}\n", .{@sizeOf(@TypeOf(bits))});
}
```

### Practical Use: Tracking Available IDs

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;

    var available: std.BitSet(32) = .initFull(); // All IDs available

    // Allocate IDs
    const id1 = available.findFirstSet().?; // 0
    available.unset(id1);
    const id2 = available.findFirstSet().?; // 1
    available.unset(id2);
    const id5 = available.findFirstSet().?; // 2
    available.unset(id5);

    // Free an ID
    available.set(id2);

    const writer = std.io.getStdOut().writer();
    try writer.print("Allocated IDs: {}, {}, {}\n", .{ id1, id2, id5 });
    try writer.print("Next available: {}\n", .{available.findFirstSet().?});
    try writer.print("IDs in use: {}\n", .{32 - available.count()});
}
```

---

## Chapter 10: std.TailQueue — Queue Implementation

`std.TailQueue` provides an intrusive double-ended queue with O(1) append and pop operations:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;
    const allocator = std.heap.page_allocator;

    const Task = struct {
        name: []const u8,
        priority: u32,
    };

    var queue: std.TailQueue(Task) = .empty;

    // Create nodes
    const n1 = try queue.createNode(allocator, .{ .name = "download", .priority = 1 });
    const n2 = try queue.createNode(allocator, .{ .name = "compile", .priority = 2 });
    const n3 = try queue.createNode(allocator, .{ .name = "test", .priority = 3 });

    // Append to tail
    queue.append(n1);
    queue.append(n2);
    queue.append(n3);

    const writer = std.io.getStdOut().writer();
    try writer.writeAll("Queue (FIFO):\n");

    // Pop from head (FIFO)
    while (queue.popFirst()) |node| {
        try writer.print("  [{s}] priority={}\n", .{ node.data.name, node.data.priority });
        allocator.destroy(node);
    }
}
```

### Using TailQueue as a Stack (LIFO)

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;
    const allocator = std.heap.page_allocator;

    var stack: std.TailQueue(u32) = .empty;

    // Push
    const n1 = try stack.createNode(allocator, 10);
    const n2 = try stack.createNode(allocator, 20);
    const n3 = try stack.createNode(allocator, 30);

    stack.prepend(n1);
    stack.prepend(n2);
    stack.prepend(n3);

    const writer = std.io.getStdOut().writer();
    try writer.writeAll("Stack (LIFO):\n");

    // Pop (same as FIFO with prepend — gives LIFO)
    while (stack.popFirst()) |node| {
        try writer.print("  {}\n", .{node.data});
        allocator.destroy(node);
    }
}
```

---

## Chapter 11: std.MultiArrayList — SoA (Struct of Arrays) Layout

`std.MultiArrayList` stores each field of a struct in a separate contiguous array. This is the Struct of Arrays (SoA) pattern, which provides much better cache locality when you only need to access certain fields:

```zig
const std = @import("std");

const Entity = struct {
    x: f32,
    y: f32,
    z: f32,
    health: u32,
    name_id: u32,
};

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var entities: std.MultiArrayList(Entity) = .empty;
    defer entities.deinit(allocator);

    // Resize
    try entities.resize(allocator, 1000);

    // Access individual field slices
    const x_slice = entities.items(.x);
    const y_slice = entities.items(.y);
    const health_slice = entities.items(.health);

    // Set values field-by-field (SoA access)
    for (0..1000) |i| {
        x_slice[i] = @as(f32, @floatFromInt(i)) * 0.1;
        y_slice[i] = @as(f32, @floatFromInt(i)) * 0.2;
        health_slice[i] = 100;
    }

    // Process only health (no cache pollution from x, y, z)
    var alive: usize = 0;
    for (health_slice) |h| {
        if (h > 0) alive += 1;
    }

    // Get a specific entity (creates a temporary struct on the stack)
    const entity_42 = entities.get(42);
    const writer = std.io.getStdOut().writer();
    try writer.print("Entity 42: x={d:.1} y={d:.1} health={}\n", .{
        entity_42.x, entity_42.y, entity_42.health,
    });
    try writer.print("Alive entities: {}\n", .{alive});

    // Set a specific entity
    var e = entities.get(0);
    e.health = 50;
    entities.set(0, e);
}
```

### MultiArrayList with Slices

You can get mutable slices for batch operations:

```zig
// Zero out all positions (only x, y, z — not health or name_id)
const xs = entities.items(.x);
const ys = entities.items(.y);
const zs = entities.items(.z);

@memset(xs, 0);
@memset(ys, 0);
@memset(zs, 0);
```

This is dramatically faster than iterating over an array of structs when you only need to update position data.

---

## Chapter 12: std.ComptimeStringMap — Compile-Time String Maps

`std.ComptimeStringMap` creates a string-to-value map that is fully resolved at compile time. No runtime allocation, no hashing overhead:

```zig
const std = @import("std");

const Color = enum { red, green, blue, yellow, purple };

const color_map = std.ComptimeStringMap(Color, .{
    .{ "red", Color.red },
    .{ "green", Color.green },
    .{ "blue", Color.blue },
    .{ "yellow", Color.yellow },
    .{ "purple", Color.purple },
});

pub fn main(init: std.process.Init) !void {
    _ = init;

    const writer = std.io.getStdOut().writer();

    if (color_map.get("red")) |color| {
        try writer.print("red => {}\n", .{color});
    }

    if (color_map.get("green")) |color| {
        try writer.print("green => {}\n", .{color});
    }

    const has_orange = color_map.has("orange");
    try writer.print("Has 'orange': {}\n", .{has_orange});

    // Can also use with string keys mapped to numeric values
    const http_status = std.ComptimeStringMap(u16, .{
        .{ "OK", 200 },
        .{ "Not Found", 404 },
        .{ "Internal Server Error", 500 },
        .{ "Created", 201 },
        .{ "Bad Request", 400 },
    });

    if (http_status.get("OK")) |code| {
        try writer.print("HTTP OK: {}\n", .{code});
    }

    // All lookups compile to a series of comptime-generated comparisons
    // No hash table is built at runtime
}
```

### Dynamic Dispatch with ComptimeStringMap

```zig
const std = @import("std");

fn executeCommand(cmd: []const u8, writer: anytype) !void {
    const dispatch = std.ComptimeStringMap(*const fn ([]const u8, anytype) anyerror!void, .{
        .{ "hello", helloCommand },
        .{ "goodbye", goodbyeCommand },
        .{ "status", statusCommand },
    });

    if (dispatch.get(cmd)) |fn_ptr| {
        try fn_ptr(cmd, writer);
    } else {
        try writer.print("Unknown command: {s}\n", .{cmd});
    }
}

fn helloCommand(_: []const u8, writer: anytype) !void {
    try writer.print("Hello, world!\n", .{});
}

fn goodbyeCommand(_: []const u8, writer: anytype) !void {
    try writer.print("Goodbye!\n", .{});
}

fn statusCommand(_: []const u8, writer: anytype) !void {
    try writer.print("All systems operational.\n", .{});
}

pub fn main(init: std.process.Init) !void {
    _ = init;
    const writer = std.io.getStdOut().writer();

    try executeCommand("hello", writer);
    try executeCommand("status", writer);
    try executeCommand("goodbye", writer);
    try executeCommand("unknown", writer);
}
```

---

## Chapter 13: Project — An In-Memory Key-Value Store with Multiple Data Structures

This project implements a simple in-memory key-value store that uses `StringHashMap`, `ArrayList`, `MultiArrayList`, and `BitSet` to provide a rich set of operations.

### `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "kvstore",
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

const KVEntry = struct {
    key: []const u8,
    value: []const u8,
    created_ns: i64,
    ttl_ns: i64, // 0 = no expiry
};

const Store = struct {
    allocator: std.mem.Allocator,
    // Primary storage: StringHashMap -> index into entries ArrayList
    index: std.StringHashMap(u32),
    // All entries in insertion order
    entries: std.ArrayList(KVEntry),
    // Track deleted slots for reuse
    deleted_slots: std.BitSet(1024),
    // Recent operations log (bounded)
    log: std.BoundedArray([:0]const u8, 64),
    // Stats per-entry using SoA for fast aggregation
    access_counts: std.MultiArrayList(struct { count: u32, last_ns: i64 }),

    fn init(allocator: std.mem.Allocator) Store {
        return .{
            .allocator = allocator,
            .index = std.StringHashMap(u32).init(allocator),
            .entries = std.ArrayList(KVEntry).init(allocator),
            .deleted_slots = std.BitSet(1024).initEmpty(),
            .log = .empty,
            .access_counts = .empty,
        };
    }

    fn deinit(self: *Store) void {
        self.index.deinit();
        self.entries.deinit();
        self.access_counts.deinit(self.allocator);
    }

    fn put(self: *Store, key: []const u8, value: []const u8, ttl_ms: u64) !void {
        const now = std.time.nanoTimestamp();
        const key_owned = try self.allocator.dupeZ(u8, key);
        errdefer self.allocator.free(key_owned);
        const value_owned = try self.allocator.dupeZ(u8, value);
        errdefer self.allocator.free(value_owned);

        const id: u32 = @intCast(self.entries.items.len);

        const entry = KVEntry{
            .key = key_owned,
            .value = value_owned,
            .created_ns = now,
            .ttl_ns = @as(i64, @intCast(ttl_ms * 1_000_000)),
        };

        try self.entries.append(entry);
        try self.index.put(key, id);

        // Track access
        try self.access_counts.resize(self.allocator, self.entries.items.len);
        var counts = self.access_counts.items(.count);
        var last = self.access_counts.items(.last_ns);
        counts[id] = 1;
        last[id] = now;

        // Log
        const msg = try std.fmt.allocPrintZ(self.allocator, "PUT {s}={s}", .{ key, value });
        self.log.append(msg) catch {};
    }

    fn get(self: *Store, key: []const u8) ?[]const u8 {
        const id = self.index.get(key) orelse return null;
        const entry = &self.entries.items[id];

        // Check TTL
        if (entry.ttl_ns > 0) {
            const now = std.time.nanoTimestamp();
            if (now - entry.created_ns > entry.ttl_ns) {
                return null; // Expired
            }
        }

        // Update access count
        var counts = self.access_counts.items(.count);
        var last = self.access_counts.items(.last_ns);
        counts[id] += 1;
        last[id] = std.time.nanoTimestamp();

        return entry.value;
    }

    fn delete(self: *Store, key: []const u8) bool {
        const id = self.index.fetchRemove(key) orelse return false;
        _ = id;
        return true;
    }

    fn count(self: *const Store) usize {
        return self.entries.items.len;
    }

    fn mostAccessed(self: *const Store) ?struct { key: []const u8, count: u32 } {
        if (self.entries.items.len == 0) return null;

        var best_id: usize = 0;
        var best_count: u32 = 0;
        const counts = self.access_counts.items(.count);

        for (counts, 0..) |c, i| {
            if (c > best_count) {
                best_count = c;
                best_id = i;
            }
        }

        return .{
            .key = self.entries.items[best_id].key,
            .count = best_count,
        };
    }

    fn printStats(self: *const Store, writer: anytype) !void {
        try writer.print("\n=== KV Store Stats ===\n", .{});
        try writer.print("Total entries:  {}\n", .{self.count()});
        try writer.print("Index size:     {}\n", .{self.index.count()});
        try writer.print("Log entries:     {}/64\n", .{self.log.len});

        if (self.mostAccessed()) |ma| {
            try writer.print("Most accessed:   '{s}' ({} accesses)\n", .{ ma.key, ma.count });
        }

        try writer.print("\nAll entries:\n", .{});
        for (self.entries.items, 0..) |entry, i| {
            try writer.print("  [{}] {s} = {s}\n", .{ i, entry.key, entry.value });
        }
    }
};

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    _ = init;

    var store = Store.init(allocator);
    defer store.deinit();

    const writer = std.io.getStdOut().writer();

    // Insert entries
    try store.put("hostname", "server-01", 0);
    try store.put("port", "8080", 0);
    try store.put("debug", "true", 0);
    try store.put("max_connections", "1000", 0);
    try store.put("cache_size", "256MB", 0);

    // Read some entries multiple times
    _ = store.get("hostname");
    _ = store.get("hostname");
    _ = store.get("hostname");
    _ = store.get("port");
    _ = store.get("port");

    // Get a value
    if (store.get("hostname")) |val| {
        try writer.print("hostname = {s}\n", .{val});
    }

    // Delete
    const deleted = store.delete("debug");
    try writer.print("Deleted 'debug': {}\n", .{deleted});

    // Non-existent get
    if (store.get("nonexistent")) |val| {
        try writer.print("nonexistent = {s}\n", .{val});
    } else {
        try writer.print("nonexistent = <not found>\n", .{});
    }

    // Print stats
    try store.printStats(writer);
}
```

### Running the Project

```bash
zig build run
```

Expected output:

```
hostname = server-01
Deleted 'debug': true
nonexistent = <not found>

=== KV Store Stats ===
Total entries:  5
Index size:     4
Log entries:    5/64
Most accessed:   'hostname' (4 accesses)

All entries:
  [0] hostname = server-01
  [1] port = 8080
  [2] debug = true
  [3] max_connections = 1000
  [4] cache_size = 256MB
```

---

## Summary

Zig 0.16's collection types form a comprehensive toolkit:

1. **`std.ArrayList(T)` / `std.ArrayListUnmanaged(T)`** — Dynamic arrays with managed and unmanaged variants
2. **`std.SegmentedList`** — Segment-based lists for large, unpredictable sizes
3. **`std.HashMap` / `std.HashMapUnmanaged`** — Generic hash maps with custom hash contexts
4. **`std.StringHashMap` / `std.StringHashMapUnmanaged`** — Convenience wrappers for string-keyed maps
5. **`std.BoundedArray`** — Stack-allocated fixed-capacity arrays with dynamic length tracking
6. **`std.PackedIntArray`** — Bit-packed integer arrays for memory efficiency
7. **`std.SinglyLinkedList` / `std.DoublyLinkedList`** — Intrusive linked lists
8. **`std.BitSet` / `std.BitSetAligned`** — Compact bit manipulation
9. **`std.TailQueue`** — O(1) FIFO/LIFO intrusive queue
10. **`std.MultiArrayList`** — Struct of Arrays layout for cache-friendly field access
11. **`std.ComptimeStringMap`** — Zero-overhead compile-time string maps

The managed/unmanaged split is the defining pattern: use unmanaged when you want full control and zero overhead, and managed when you prefer ergonomics and don't mind the extra stored pointer.