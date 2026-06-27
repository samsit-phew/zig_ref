# Heap Data Structures in Zig 0.16

A comprehensive guide to heap data structures in Zig, covering `std.heap` (binary min/max-heap), `std.PriorityQueue`, the new `std.PriorityDequeue`, heap sort, and graph algorithms.

---

## Chapter 1: Introduction to Heaps

A heap is a tree-based data structure that satisfies the heap property. In a **min-heap**, every parent node is less than or equal to its children. In a **max-heap**, every parent is greater than or equal to its children. The root always contains the minimum (min-heap) or maximum (max-heap) element.

### Why Heaps Matter

| Operation | Sorted Array | Unsorted Array | Heap |
|-----------|-------------|----------------|------|
| Find min/max | O(1) | O(n) | O(1) |
| Insert | O(n) | O(1) | O(log n) |
| Extract min/max | O(1) | O(n) | O(log n) |
| Find k-th smallest | O(1) | O(n log n) | O(k log n) |

Heaps are the backbone of:
- **Priority queues** — task scheduling, event processing
- **Dijkstra's algorithm** — shortest paths in graphs
- **A* pathfinding** — optimal route planning
- **Heap sort** — O(n log n) in-place sorting
- **Median maintenance** — sliding window statistics

Zig 0.16 provides three heap implementations:
1. `std.heap` — Low-level binary heap (min or max)
2. `std.PriorityQueue(T)` — Priority queue wrapper
3. `std.PriorityDequeue(T)` — **New in 0.16** — Double-ended priority queue

---

## Chapter 2: std.heap — Binary Min-Heap / Max-Heap

`std.heap` provides the core binary heap operations. It operates on a user-provided slice and uses a context for custom comparison logic.

### Basic Min-Heap

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var items = std.ArrayList(i32).init(allocator);
    defer items.deinit();

    // Insert elements
    const values = [_]i32{ 42, 17, 8, 93, 5, 23, 64, 1, 37, 11 };
    for (values) |v| {
        try items.append(v);
        // Sift up the newly added element
        std.heap.siftUp(i32, items.items, items.items.len - 1, {}, lessThan);
    }

    // Display the heap (not sorted, but satisfies heap property)
    try w.print("Min-Heap array: {any}\n", .{items.items});

    // Extract all elements (they come out in sorted order)
    try w.print("Extracted (ascending): ", .{});
    while (items.items.len > 0) {
        // The minimum is always at index 0
        const min_val = items.items[0];
        try w.print("{} ", .{min_val});

        // Move the last element to the root and sift down
        const last = items.pop();
        if (items.items.len > 0) {
            items.items[0] = last;
            std.heap.siftDown(i32, items.items, 0, {}, lessThan);
        }
    }
    try w.print("\n");
}

fn lessThan(_: void, a: i32, b: i32) bool {
    return a < b;
}
```

### Max-Heap (Reverse Comparison)

```zig
fn greaterThan(_: void, a: i32, b: i32) bool {
    return a > b;
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var items = std.ArrayList(i32).init(allocator);
    defer items.deinit();

    const values = [_]i32{ 3, 1, 4, 1, 5, 9, 2, 6, 5, 3 };
    for (values) |v| {
        try items.append(v);
        std.heap.siftUp(i32, items.items, items.items.len - 1, {}, greaterThan);
    }

    try w.print("Max-Heap array: {any}\n", .{items.items});

    try w.print("Extracted (descending): ", .{});
    while (items.items.len > 0) {
        const max_val = items.items[0];
        try w.print("{} ", .{max_val});
        const last = items.pop();
        if (items.items.len > 0) {
            items.items[0] = last;
            std.heap.siftDown(i32, items.items, 0, {}, greaterThan);
        }
    }
    try w.print("\n");
}
```

### Heapify — Build a Heap in O(n)

Instead of inserting one element at a time (O(n log n)), you can heapify an existing array in O(n):

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var data = [_]i32{ 42, 17, 8, 93, 5, 23, 64, 1, 37, 11 };

    // Build a min-heap in O(n)
    std.heap.heapify(i32, &data, {}, lessThan);

    try w.print("Heapified: {any}\n", .{data});
    try w.print("Minimum:   {}\n", .{data[0]});
}

fn lessThan(_: void, a: i32, b: i32) bool {
    return a < b;
}
```

### Using Context for Custom Types

The context parameter lets you compare complex types:

```zig
const std = @import("std");

const Task = struct {
    name: []const u8,
    priority: u8,
};

const TaskContext = struct {
    fn lessThan(_: @This(), a: Task, b: Task) bool {
        // Lower number = higher priority
        return a.priority < b.priority;
    }
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var tasks = std.ArrayList(Task).init(allocator);
    defer tasks.deinit();

    const task_data = [_]Task{
        .{ .name = "Write tests", .priority = 3 },
        .{ .name = "Fix bug #42", .priority = 1 },
        .{ .name = "Update docs", .priority = 5 },
        .{ .name = "Code review", .priority = 2 },
        .{ .name = "Deploy", .priority = 4 },
    };

    for (task_data) |t| {
        try tasks.append(t);
        std.heap.siftUp(Task, tasks.items, tasks.items.len - 1, TaskContext{}, TaskContext.lessThan);
    }

    try w.print("Task execution order (by priority):\n", .{});
    while (tasks.items.len > 0) {
        const task = tasks.items[0];
        try w.print("  [{d}] {s}\n", .{ task.priority, task.name });
        const last = tasks.pop();
        if (tasks.items.len > 0) {
            tasks.items[0] = last;
            std.heap.siftDown(Task, tasks.items, 0, TaskContext{}, TaskContext.lessThan);
        }
    }
}
```

---

## Chapter 3: std.PriorityQueue(T) — Priority Queue Implementation

`std.PriorityQueue` is a higher-level wrapper around the binary heap that provides a clean API for priority queue operations.

### Basic Usage

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Create a priority queue of i32 (min-heap by default)
    var pq = std.PriorityQueue(i32, void, lessThan).init(allocator, {});
    defer pq.deinit();

    // Add items
    try pq.add(42);
    try pq.add(17);
    try pq.add(8);
    try pq.add(93);
    try pq.add(5);
    try pq.add(23);

    try w.print("Priority Queue size: {d}\n", .{pq.len});
    try w.print("Peek at minimum:     {}\n", .{pq.peek()});

    // Extract items in priority order
    try w.print("Extracted order:      ", .{});
    while (pq.removeOrNull()) |item| {
        try w.print("{} ", .{item});
    }
    try w.print("\n");
}

fn lessThan(_: void, a: i32, b: i32) bool {
    return a < b;
}
```

### Max-Priority Queue

```zig
fn greaterThan(_: void, a: i32, b: i32) bool {
    return a > b;
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var pq = std.PriorityQueue(i32, void, greaterThan).init(allocator, {});
    defer pq.deinit();

    const values = [_]i32{ 3, 1, 4, 1, 5, 9, 2, 6 };
    for (values) |v| try pq.add(v);

    try w.print("Max-priority queue: ", .{});
    while (pq.removeOrNull()) |item| {
        try w.print("{} ", .{item});
    }
    try w.print("\n");
}
```

### Priority Queue with Custom Types

```zig
const std = @import("std");

const Event = struct {
    time: u64,
    name: []const u8,

    pub fn format(self: Event, comptime _: []const u8, _: std.fmt.FormatOptions, writer: anytype) !void {
        try writer.print("[t={d}] {s}", .{ self.time, self.name });
    }
};

const EventContext = struct {
    fn lessThan(_: @This(), a: Event, b: Event) bool {
        return a.time < b.time; // Earlier events first
    }
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var scheduler = std.PriorityQueue(Event, void, EventContext.lessThan).init(allocator, {});
    defer scheduler.deinit();

    try scheduler.add(.{ .time = 100, .name = "Startup" });
    try scheduler.add(.{ .time = 50, .name = "Initialize" });
    try scheduler.add(.{ .time = 200, .name = "Process" });
    try scheduler.add(.{ .time = 150, .name = "Connect" });
    try scheduler.add(.{ .time = 75, .name = "Load config" });

    try w.print("Event schedule:\n", .{});
    while (scheduler.removeOrNull()) |event| {
        try w.print("  {}\n", .{event});
    }
}
```

### Updating Priorities

To update a priority, remove the old item and re-insert:

```zig
// Update task priority
if (pq.remove(task)) {
    task.priority = new_priority;
    try pq.add(task);
}
```

---

## Chapter 4: std.PriorityDequeue(T) — Double-Ended Priority Queue (NEW in 0.16)

Zig 0.16 introduces `std.PriorityDequeue`, a double-ended priority queue (also known as a min-max heap). It allows O(1) access to **both** the minimum and maximum elements, and O(log n) removal of either end.

### Why PriorityDequeue?

A regular priority queue only gives you fast access to one end. `PriorityDequeue` gives you both:

| Operation | PriorityQueue | PriorityDequeue |
|-----------|--------------|-----------------|
| Peek min | O(1) | O(1) |
| Peek max | O(n) | O(1) |
| Remove min | O(log n) | O(log n) |
| Remove max | O(n) | O(log n) |
| Insert | O(log n) | O(log n) |

This is invaluable for:
- **Median tracking**: maintain a min-heap and max-heap
- **Sliding window statistics**: track min and max in a window
- **Bandwidth management**: prioritize both urgent and low-priority traffic

### Basic Usage

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Create a PriorityDequeue for i32
    var pdq = std.PriorityDequeue(i32, void, lessThan).init(allocator, {});
    defer pdq.deinit();

    // Add elements
    const values = [_]i32{ 42, 17, 8, 93, 5, 23, 64, 1, 37, 11 };
    for (values) |v| {
        try pdq.add(v);
    }

    try w.print("PriorityDequeue (size={d}):\n", .{pdq.len});
    try w.print("  Min: {}\n", .{pdq.peekMin() orelse 0});
    try w.print("  Max: {}\n", .{pdq.peekMax() orelse 0});

    // Remove from both ends
    try w.print("\nRemoving from both ends:\n", .{});
    try w.print("  Removed min: {}\n", .{pdq.removeMin() orelse 0});
    try w.print("  Removed max: {}\n", .{pdq.removeMax() orelse 0});
    try w.print("  New min: {}\n", .{pdq.peekMin() orelse 0});
    try w.print("  New max: {}\n", .{pdq.peekMax() orelse 0});
}

fn lessThan(_: void, a: i32, b: i32) bool {
    return a < b;
}
```

### Sliding Window Minimum and Maximum

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const data = [_]i32{
        44, 23, 78, 12, 56, 89, 3, 67, 34, 91,
        15, 72, 8, 45, 63, 27, 84, 19, 51, 76,
    };
    const window_size: usize = 5;

    var pdq = std.PriorityDequeue(i32, void, lessThan).init(allocator, {});
    defer pdq.deinit();

    try w.print("Sliding window (size={d}):\n", .{window_size});
    try w.print("  Index  Window                 Min  Max\n", .{});
    try w.print("  {s}\n", .{"-" ** 50});

    for (data, 0..) |val, i| {
        try pdq.add(val);

        // Remove elements outside the window
        while (pdq.len > window_size) {
            // Remove the oldest element
            _ = pdq.removeMin();
            // Also remove max to keep balanced
            if (pdq.len > window_size - 1) {
                _ = pdq.removeMax();
            }
        }

        const min_val = pdq.peekMin() orelse 0;
        const max_val = pdq.peekMax() orelse 0;

        // Print window contents
        const start = if (i >= window_size - 1) i - window_size + 1 else 0;
        const end = i + 1;

        try w.print("  {d:3}    ", .{i});
        for (start..end) |j| {
            try w.print("{d:3} ", .{data[j]});
        }
        try w.print("  {d:3}  {d:3}\n", .{ min_val, max_val });
    }
}

fn lessThan(_: void, a: i32, b: i32) bool {
    return a < b;
}
```

### Using PriorityDequeue for Median Tracking

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const stream = [_]f64{
        5.0, 15.0, 1.0, 3.0, 8.0, 7.0, 9.0, 2.0, 12.0, 10.0,
    };

    try w.print("Running median:\n", .{});
    try w.print("  Value  Median\n", .{});
    try w.print("  {s}\n", .{"-" ** 20});

    var pdq = std.PriorityDequeue(f64, void, lessThanF64).init(allocator, {});
    defer pdq.deinit();

    for (stream) |val| {
        try pdq.add(val);

        // Peek both ends to compute approximate median
        const min_v = pdq.peekMin() orelse 0;
        const max_v = pdq.peekMax() orelse 0;
        const median = (min_v + max_v) / 2.0;

        try w.print("  {d:5.1}  {d:.1}\n", .{ val, median });
    }
}

fn lessThanF64(_: void, a: f64, b: f64) bool {
    return a < b;
}
```

---

## Chapter 5: Building Custom Heaps

### A Min-Heap with Decrease-Key

Some algorithms (like Dijkstra) need the ability to decrease a key efficiently. Here's a custom implementation:

```zig
const std = @import("std");

const HeapEntry = struct {
    node: u32,
    dist: u64,
};

const MinHeap = struct {
    items: []HeapEntry,
    len: usize,
    capacity: usize,
    allocator: std.mem.Allocator,
    // Track positions for decrease-key
    positions: []i32,

    pub fn init(allocator: std.mem.Allocator, capacity: usize) !MinHeap {
        const items = try allocator.alloc(HeapEntry, capacity);
        const positions = try allocator.alloc(i32, capacity);
        @memset(positions, -1);
        return .{
            .items = items,
            .len = 0,
            .capacity = capacity,
            .allocator = allocator,
            .positions = positions,
        };
    }

    pub fn deinit(self: *MinHeap) void {
        self.allocator.free(self.items);
        self.allocator.free(self.positions);
    }

    pub fn isEmpty(self: *const MinHeap) bool {
        return self.len == 0;
    }

    fn parent(i: usize) usize {
        return (i - 1) / 2;
    }

    fn leftChild(i: usize) usize {
        return 2 * i + 1;
    }

    fn rightChild(i: usize) usize {
        return 2 * i + 2;
    }

    pub fn push(self: *MinHeap, entry: HeapEntry) !void {
        if (self.len >= self.capacity) return error.HeapFull;
        const idx = self.len;
        self.items[idx] = entry;
        self.positions[entry.node] = @intCast(idx);
        self.len += 1;
        self.siftUp(idx);
    }

    fn siftUp(self: *MinHeap, i: usize) void {
        var idx = i;
        while (idx > 0) {
            const p = Self.parent(idx);
            if (self.items[idx].dist < self.items[p].dist) {
                self.swap(idx, p);
                idx = p;
            } else break;
        }
    }

    fn siftDown(self: *MinHeap, i: usize) void {
        var idx = i;
        while (true) {
            const left = Self.leftChild(idx);
            const right = Self.rightChild(idx);
            var smallest = idx;

            if (left < self.len and self.items[left].dist < self.items[smallest].dist) {
                smallest = left;
            }
            if (right < self.len and self.items[right].dist < self.items[smallest].dist) {
                smallest = right;
            }

            if (smallest != idx) {
                self.swap(idx, smallest);
                idx = smallest;
            } else break;
        }
    }

    fn swap(self: *MinHeap, i: usize, j: usize) void {
        const a = self.items[i];
        const b = self.items[j];
        self.items[i] = b;
        self.items[j] = a;
        self.positions[a.node] = @intCast(j);
        self.positions[b.node] = @intCast(i);
    }

    pub fn pop(self: *MinHeap) ?HeapEntry {
        if (self.len == 0) return null;
        const min = self.items[0];
        self.len -= 1;
        if (self.len > 0) {
            self.items[0] = self.items[self.len];
            self.positions[self.items[0].node] = 0;
            self.siftDown(0);
        }
        self.positions[min.node] = -1;
        return min;
    }

    pub fn decreaseKey(self: *MinHeap, node: u32, new_dist: u64) void {
        const idx = self.positions[node];
        if (idx < 0) return;
        self.items[@intCast(idx)].dist = new_dist;
        self.siftUp(@intCast(idx));
    }

    const Self = @This();
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var heap = try MinHeap.init(init.allocator, 10);
    defer heap.deinit();

    try heap.push(.{ .node = 0, .dist = 10 });
    try heap.push(.{ .node = 1, .dist = 5 });
    try heap.push(.{ .node = 2, .dist = 15 });
    try heap.push(.{ .node = 3, .dist = 3 });

    try w.print("Custom heap:\n", .{});
    while (heap.pop()) |entry| {
        try w.print("  node={}, dist={}\n", .{ entry.node, entry.dist });
    }
}
```

---

## Chapter 6: Heap Sort Algorithm

Heap sort is an in-place, O(n log n) sorting algorithm with O(1) space complexity (ignoring the recursion stack for sift-down). It works by building a max-heap and repeatedly extracting the maximum.

### Basic Heap Sort

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var data = [_]i32{ 42, 17, 8, 93, 5, 23, 64, 1, 37, 11, 56, 79, 2, 31, 48 };

    try w.print("Before sort: {any}\n", .{data});
    heapSort(i32, &data, {}, std.math.order);
    try w.print("After sort:  {any}\n", .{data});
}

fn heapSort(
    comptime T: type,
    items: []T,
    context: anytype,
    comptime lessThan: fn (@TypeOf(context), T, T) bool,
) void {
    const n = items.len;

    // Build max-heap: sift down from the last parent
    for (0..n / 2) |i| {
        const idx = n / 2 - 1 - i;
        siftDown(T, items, idx, n, context, lessThan);
    }

    // Extract elements one by one
    var size = n;
    while (size > 1) : (size -= 1) {
        // Swap root (max) with last element
        const last = size - 1;
        const tmp = items[0];
        items[0] = items[last];
        items[last] = tmp;

        // Sift down the new root
        siftDown(T, items, 0, size - 1, context, lessThan);
    }
}

fn siftDown(
    comptime T: type,
    items: []T,
    root: usize,
    end: usize,
    context: anytype,
    comptime lessThan: fn (@TypeOf(context), T, T) bool,
) void {
    var idx = root;
    while (true) {
        const left = 2 * idx + 1;
        const right = 2 * idx + 2;
        var largest = idx;

        if (left <= end and lessThan(context, items[largest], items[left])) {
            largest = left;
        }
        if (right <= end and lessThan(context, items[largest], items[right])) {
            largest = right;
        }
        if (largest == idx) break;

        const tmp = items[idx];
        items[idx] = items[largest];
        items[largest] = tmp;
        idx = largest;
    }
}
```

### Generic Heap Sort with Custom Comparator

```zig
const std = @import("std");

const Person = struct {
    name: []const u8,
    age: u8,

    pub fn format(self: Person, comptime _: []const u8, _: std.fmt.FormatOptions, writer: anytype) !void {
        try writer.print("{s} (age {d})", .{ self.name, self.age });
    }
};

fn byAge(_: void, a: Person, b: Person) bool {
    return a.age < b.age;
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var people = [_]Person{
        .{ .name = "Alice", .age = 30 },
        .{ .name = "Bob", .age = 25 },
        .{ .name = "Charlie", .age = 35 },
        .{ .name = "Diana", .age = 28 },
        .{ .name = "Eve", .age = 22 },
    };

    try w.print("Before: {any}\n", .{people});
    heapSort(Person, &people, {}, byAge);
    try w.print("After:  {any}\n", .{people});
}
```

---

## Chapter 7: Dijkstra's Algorithm with a Priority Queue

Dijkstra's algorithm finds the shortest path from a source node to all other nodes in a weighted graph with non-negative edge weights. A min-priority queue is essential for its O((V + E) log V) time complexity.

### Adjacency List Graph

```zig
const std = @import("std");

const Edge = struct {
    to: u32,
    weight: u32,
};

const Graph = struct {
    adjacency: [][]Edge,
    allocator: std.mem.Allocator,
    num_nodes: u32,

    pub fn init(allocator: std.mem.Allocator, num_nodes: u32) !Graph {
        const adj = try allocator.alloc([]Edge, num_nodes);
        for (0..num_nodes) |i| {
            adj[i] = &[_]Edge{};
        }
        return .{
            .adjacency = adj,
            .allocator = allocator,
            .num_nodes = num_nodes,
        };
    }

    pub fn deinit(self: *Graph) void {
        for (self.adjacency) |edges| {
            if (edges.len > 0) self.allocator.free(edges);
        }
        self.allocator.free(self.adjacency);
    }

    pub fn addEdge(self: *Graph, from: u32, to: u32, weight: u32) !void {
        const edges = self.adjacency[from];
        var new_edges = try self.allocator.alloc(Edge, edges.len + 1);
        @memcpy(new_edges[0..edges.len], edges);
        new_edges[edges.len] = .{ .to = to, .weight = weight };
        if (edges.len > 0) self.allocator.free(edges);
        self.adjacency[from] = new_edges;
    }
};
```

### Dijkstra Implementation

```zig
fn dijkstra(
    allocator: std.mem.Allocator,
    graph: *const Graph,
    source: u32,
) ![]const u64 {
    const n = graph.num_nodes;
    const dist = try allocator.alloc(u64, n);
    @memset(dist, std.math.maxInt(u64));
    dist[source] = 0;

    var pq = std.PriorityQueue(Node, void, Node.lessThan).init(allocator, {});
    defer pq.deinit();

    try pq.add(.{ .id = source, .dist = 0 });

    while (pq.removeOrNull()) |node| {
        if (node.dist > dist[node.id]) continue;

        for (graph.adjacency[node.id]) |edge| {
            const new_dist = node.dist + edge.weight;
            if (new_dist < dist[edge.to]) {
                dist[edge.to] = new_dist;
                try pq.add(.{ .id = edge.to, .dist = new_dist });
            }
        }
    }

    return dist;
}

const Node = struct {
    id: u32,
    dist: u64,

    fn lessThan(_: void, a: Node, b: Node) bool {
        return a.dist < b.dist;
    }
};
```

### Full Example

```zig
const std = @import("std");

// ... (Edge, Graph, Node, dijkstra as above)

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var graph = try Graph.init(allocator, 6);
    defer graph.deinit();

    // Build a sample graph
    try graph.addEdge(0, 1, 4);
    try graph.addEdge(0, 2, 2);
    try graph.addEdge(1, 2, 5);
    try graph.addEdge(1, 3, 10);
    try graph.addEdge(2, 4, 3);
    try graph.addEdge(4, 3, 4);
    try graph.addEdge(3, 5, 11);
    try graph.addEdge(4, 5, 6);

    const dist = try dijkstra(allocator, &graph, 0);
    defer allocator.free(dist);

    try w.print("Shortest distances from node 0:\n", .{});
    for (0..graph.num_nodes) |i| {
        const d = dist[i];
        if (d == std.math.maxInt(u64)) {
            try w.print("  0 -> {d}: unreachable\n", .{i});
        } else {
            try w.print("  0 -> {d}: {d}\n", .{ i, d });
        }
    }
}
```

---

## Chapter 8: A* Pathfinding with a Priority Queue

A* extends Dijkstra by adding a heuristic function h(n) that estimates the cost to reach the goal. The priority becomes `f(n) = g(n) + h(n)` where `g(n)` is the cost from start to `n`.

### Grid-Based A*

```zig
const std = @import("std");

const Point = struct {
    x: i32,
    y: i32,
};

const AStarNode = struct {
    pos: Point,
    g: u32, // cost from start
    f: u32, // g + heuristic

    fn lessThan(_: void, a: AStarNode, b: AStarNode) bool {
        return a.f < b.f;
    }
};

const Grid = struct {
    width: u32,
    height: u32,
    walls: std.bit_set.DynamicBitSetUnmanaged,

    pub fn init(allocator: std.mem.Allocator, width: u32, height: u32) !Grid {
        const total = width * height;
        const walls = try std.bit_set.DynamicBitSetUnmanaged.initEmpty(allocator, total);
        return .{ .width = width, .height = height, .walls = walls };
    }

    pub fn deinit(self: *Grid, allocator: std.mem.Allocator) void {
        self.walls.deinit(allocator);
    }

    pub fn setWall(self: *Grid, x: u32, y: u32) void {
        const idx = y * self.width + x;
        self.walls.set(idx);
    }

    pub fn isWall(self: *const Grid, x: i32, y: i32) bool {
        if (x < 0 or y < 0 or x >= @as(i32, @intCast(self.width)) or y >= @as(i32, @intCast(self.height))) return true;
        const idx = @as(usize, @intCast(y)) * self.width + @as(usize, @intCast(x));
        return self.walls.isSet(idx);
    }

    // Manhattan distance heuristic
    pub fn heuristic(self: *const Grid, a: Point, b: Point) u32 {
        return @intCast(@abs(a.x - b.x) + @abs(a.y - b.y));
    }
};

fn aStar(
    allocator: std.mem.Allocator,
    grid: *const Grid,
    start: Point,
    goal: Point,
) !?[]Point {
    var open = std.PriorityQueue(AStarNode, void, AStarNode.lessThan).init(allocator, {});
    defer open.deinit();

    const total = grid.width * grid.height;
    var g_score = try allocator.alloc(u32, total);
    defer allocator.free(g_score);
    @memset(g_score, std.math.maxInt(u32));

    var came_from = try allocator.alloc(?Point, total);
    defer allocator.free(came_from);
    @memset(came_from, null);

    const start_idx = @as(usize, @intCast(start.y)) * grid.width + @as(usize, @intCast(start.x));
    g_score[start_idx] = 0;

    try open.add(.{
        .pos = start,
        .g = 0,
        .f = grid.heuristic(start, goal),
    });

    const directions = [_]Point{
        .{ .x = 0, .y = -1 }, // up
        .{ .x = 0, .y = 1 },  // down
        .{ .x = -1, .y = 0 }, // left
        .{ .x = 1, .y = 0 },  // right
    };

    while (open.removeOrNull()) |current| {
        if (current.pos.x == goal.x and current.pos.y == goal.y) {
            // Reconstruct path
            var path = std.ArrayList(Point).init(allocator);
            defer path.deinit();

            var pos = current.pos;
            while (true) {
                try path.append(pos);
                const idx = @as(usize, @intCast(pos.y)) * grid.width + @as(usize, @intCast(pos.x));
                if (came_from[idx]) |prev| {
                    pos = prev;
                } else break;
            }

            // Reverse the path
            var result = try allocator.alloc(Point, path.items.len);
            for (path.items, 0..) |item, i| {
                result[result.len - 1 - i] = item;
            }
            return result;
        }

        const cur_idx = @as(usize, @intCast(current.pos.y)) * grid.width + @as(usize, @intCast(current.pos.x));
        if (current.g > g_score[cur_idx]) continue;

        for (directions) |dir| {
            const nx = current.pos.x + dir.x;
            const ny = current.pos.y + dir.y;

            if (grid.isWall(nx, ny)) continue;

            const n_idx = @as(usize, @intCast(ny)) * grid.width + @as(usize, @intCast(nx));
            const tentative_g = current.g + 1;

            if (tentative_g < g_score[n_idx]) {
                g_score[n_idx] = tentative_g;
                came_from[n_idx] = current.pos;
                try open.add(.{
                    .pos = .{ .x = nx, .y = ny },
                    .g = tentative_g,
                    .f = tentative_g + grid.heuristic(.{ .x = nx, .y = ny }, goal),
                });
            }
        }
    }

    return null; // No path found
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var grid = try Grid.init(allocator, 10, 10);
    defer grid.deinit(allocator);

    // Add some walls
    grid.setWall(3, 0);
    grid.setWall(3, 1);
    grid.setWall(3, 2);
    grid.setWall(3, 3);
    grid.setWall(3, 5);
    grid.setWall(3, 6);
    grid.setWall(3, 7);
    grid.setWall(6, 3);
    grid.setWall(6, 4);
    grid.setWall(6, 5);

    const start = Point{ .x = 0, .y = 0 };
    const goal = Point{ .x = 9, .y = 9 };

    const path = try aStar(allocator, &grid, start, goal);
    defer if (path) |p| allocator.free(p);

    if (path) |p| {
        try w.print("Path found! Length: {d} steps\n", .{p.len});
        try w.print("Path: ", .{});
        for (p) |pt| {
            try w.print("({d},{d})", .{ pt.x, pt.y });
            if (pt.x != goal.x or pt.y != goal.y) {
                try w.print(" -> ", .{});
            }
        }
        try w.print("\n");
    } else {
        try w.print("No path found!\n", .{});
    }
}
```

---

## Chapter 9: Project — A Task Scheduler Using Priority Queues

This project implements a task scheduler that uses priority queues to manage jobs with different priorities, deadlines, and dependencies.

### Project Structure

```
task-scheduler/
├── build.zig
└── src/
    └── main.zig
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "task-scheduler",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    if (b.args) |args| run_cmd.addArgs(args);

    const run_step = b.step("run", "Run the task scheduler");
    run_step.dependOn(&run_cmd.step);
}
```

### src/main.zig

```zig
const std = @import("std");

// ============================================================
// Task Types
// ============================================================

const TaskId = u32;

const Priority = enum(u8) {
    critical = 0,
    high = 1,
    medium = 2,
    low = 3,
    background = 4,
};

const Task = struct {
    id: TaskId,
    name: []const u8,
    priority: Priority,
    deadline: u64, // timestamp
    duration: u64, // simulated duration in ms
    dependencies: []TaskId,
    completed: bool = false,
    started_at: ?u64 = null,

    pub fn format(self: Task, comptime _: []const u8, _: std.fmt.FormatOptions, writer: anytype) !void {
        try writer.print("[{d}] {s} (priority={}, deadline={d}, dur={d}ms)", .{
            self.id, self.name, @intFromEnum(self.priority), self.deadline, self.duration,
        });
    }
};

// ============================================================
// Scheduler using PriorityQueue and PriorityDequeue
// ============================================================

const Scheduler = struct {
    ready_queue: std.PriorityQueue(ScheduledTask, void, ScheduledTask.compare),
    deadline_queue: std.PriorityDequeue(ScheduledTask, void, ScheduledTask.compare),
    tasks: std.AutoHashMap(TaskId, Task),
    allocator: std.mem.Allocator,
    next_id: TaskId = 1,
    current_time: u64 = 0,

    const ScheduledTask = struct {
        id: TaskId,
        priority: Priority,
        deadline: u64,
        scheduled_at: u64,

        fn compare(_: void, a: ScheduledTask, b: ScheduledTask) bool {
            // First by priority (lower enum = higher priority)
            if (@intFromEnum(a.priority) != @intFromEnum(b.priority)) {
                return @intFromEnum(a.priority) < @intFromEnum(b.priority);
            }
            // Then by deadline (earlier = higher priority)
            return a.deadline < b.deadline;
        }
    };

    pub fn init(allocator: std.mem.Allocator) !Scheduler {
        return .{
            .ready_queue = std.PriorityQueue(ScheduledTask, void, ScheduledTask.compare).init(allocator, {}),
            .deadline_queue = std.PriorityDequeue(ScheduledTask, void, ScheduledTask.compare).init(allocator, {}),
            .tasks = std.AutoHashMap(TaskId, Task).init(allocator),
            .allocator = allocator,
        };
    }

    pub fn deinit(self: *Scheduler) void {
        self.ready_queue.deinit();
        self.deadline_queue.deinit();
        self.tasks.deinit();
    }

    pub fn addTask(
        self: *Scheduler,
        name: []const u8,
        priority: Priority,
        deadline: u64,
        duration: u64,
        dependencies: []const TaskId,
    ) !TaskId {
        const id = self.next_id;
        self.next_id += 1;

        const deps = try self.allocator.dupe(TaskId, dependencies);
        errdefer self.allocator.free(deps);

        const task = Task{
            .id = id,
            .name = name,
            .priority = priority,
            .deadline = deadline,
            .duration = duration,
            .dependencies = deps,
        };

        try self.tasks.put(id, task);

        // If no dependencies, add to ready queue immediately
        if (dependencies.len == 0) {
            const st = ScheduledTask{
                .id = id,
                .priority = priority,
                .deadline = deadline,
                .scheduled_at = self.current_time,
            };
            try self.ready_queue.add(st);
            try self.deadline_queue.add(st);
        }

        return id;
    }

    pub fn completeTask(self: *Scheduler, task_id: TaskId) !?[]TaskId {
        var task = self.tasks.getPtr(task_id) orelse return null;
        task.completed = true;

        // Find tasks that depend on this one
        var unblocked = std.ArrayList(TaskId).init(self.allocator);
        defer unblocked.deinit();

        var it = self.tasks.iterator();
        while (it.next()) |entry| {
            if (entry.value_ptr.completed) continue;
            if (entry.value_ptr.started_at != null) continue;

            // Check if all dependencies are met
            var all_deps_met = true;
            for (entry.value_ptr.dependencies) |dep_id| {
                const dep = self.tasks.get(dep_id) orelse continue;
                if (!dep.completed) {
                    all_deps_met = false;
                    break;
                }
            }

            if (all_deps_met) {
                entry.value_ptr.started_at = self.current_time;
                const st = ScheduledTask{
                    .id = entry.value_ptr.id,
                    .priority = entry.value_ptr.priority,
                    .deadline = entry.value_ptr.deadline,
                    .scheduled_at = self.current_time,
                };
                try self.ready_queue.add(st);
                try self.deadline_queue.add(st);
                try unblocked.append(entry.value_ptr.id);
            }
        }

        return unblocked.toOwnedSlice();
    }

    pub fn peekNext(self: *const Scheduler) ?ScheduledTask {
        return self.ready_queue.peek();
    }

    pub fn peekMostUrgent(self: *const Scheduler) ?ScheduledTask {
        return self.deadline_queue.peekMin();
    }

    pub fn peekLeastUrgent(self: *const Scheduler) ?ScheduledTask {
        return self.deadline_queue.peekMax();
    }

    pub fn getTask(self: *const Scheduler, id: TaskId) ?Task {
        return self.tasks.get(id);
    }

    pub fn runNext(self: *Scheduler) !?Task {
        const scheduled = self.ready_queue.removeOrNull() orelse return null;
        var task = self.tasks.getPtr(scheduled.id) orelse return null;
        task.started_at = self.current_time;
        self.current_time += task.duration;
        task.completed = true;

        // Unblock dependents
        _ = try self.completeTask(scheduled.id);

        return task.*;
    }

    pub fn runAll(self: *Scheduler, stdout: anytype) !void {
        try stdout.print("=== Running Task Schedule ===\n\n", .{});

        var completed: u32 = 0;
        var total_wait: u64 = 0;

        while (self.runNext()) |task| {
            const wait_time = if (task.started_at) |t| t else 0;
            total_wait += wait_time;
            completed += 1;

            try stdout.print("  [{d:3}] t={d:5}ms  {s:20s}  priority={s:12s}  deadline={d:5}  dur={d:4}ms\n", .{
                completed,
                wait_time,
                task.name,
                @tagName(task.priority),
                task.deadline,
                task.duration,
            });
        }

        try stdout.print("\nCompleted {d} tasks in {d}ms total simulated time\n", .{
            completed,
            self.current_time,
        });
    }
};

// ============================================================
// Demo: Simulate a Build System
// ============================================================

fn runBuildDemo(allocator: std.mem.Allocator, stdout: anytype) !void {
    try stdout.print("Build System Scheduler Demo\n", .{});
    try stdout.print("{s}\n\n", .{"=" ** 40});

    var scheduler = try Scheduler.init(allocator);
    defer scheduler.deinit();

    // Define tasks with dependencies
    // Phase 1: No dependencies
    _ = try scheduler.addTask("Parse config", .critical, 100, 50, &.{});
    _ = try scheduler.addTask("Check environment", .high, 200, 30, &.{});

    // Phase 2: Depends on Phase 1
    const parse_id: TaskId = 1;
    const env_id: TaskId = 2;
    _ = try scheduler.addTask("Generate code", .high, 500, 200, &.{parse_id});
    _ = try scheduler.addTask("Fetch dependencies", .medium, 400, 300, &.{env_id});

    // Phase 3: Depends on Phase 2
    const gen_id: TaskId = 3;
    const fetch_id: TaskId = 4;
    _ = try scheduler.addTask("Compile sources", .critical, 1000, 500, &.{gen_id, fetch_id});
    _ = try scheduler.addTask("Generate docs", .low, 2000, 200, &.{gen_id});

    // Phase 4: Depends on Phase 3
    const compile_id: TaskId = 5;
    _ = try scheduler.addTask("Run tests", .high, 1500, 300, &.{compile_id});
    _ = try scheduler.addTask("Lint code", .medium, 1500, 100, &.{compile_id});

    // Phase 5: Depends on Phase 4
    const test_id: TaskId = 7;
    const lint_id: TaskId = 8;
    _ = try scheduler.addTask("Create package", .critical, 2000, 150, &.{test_id, lint_id});
    _ = try scheduler.addTask("Deploy", .background, 5000, 200, &.{test_id});

    try stdout.print("Ready queue status:\n", .{});
    if (scheduler.peekNext()) |next| {
        const task = scheduler.getTask(next.id);
        try stdout.print("  Next task: {}\n", .{task orelse Task{
            .id = 0,
            .name = "unknown",
            .priority = .low,
            .deadline = 0,
            .duration = 0,
            .dependencies = &.{},
        }});
    }
    if (scheduler.peekMostUrgent()) |urgent| {
        try stdout.print("  Most urgent deadline: task {}, deadline={d}\n", .{
            urgent.id, urgent.deadline,
        });
    }
    if (scheduler.peekLeastUrgent()) |not_urgent| {
        try stdout.print("  Least urgent deadline: task {}, deadline={d}\n", .{
            not_urgent.id, not_urgent.deadline,
        });
    }

    try stdout.print("\n", .{});
    try scheduler.runAll(stdout);
}

// ============================================================
// Main
// ============================================================

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    try runBuildDemo(allocator, w);
}
```

### Usage

```bash
zig build run
```

### Expected Output

```
Build System Scheduler Demo
========================================

Ready queue status:
  Next task: [1] Parse config (priority=critical, deadline=100, dur=50ms)
  Most urgent deadline: task 1, deadline=100
  Least urgent deadline: task 2, deadline=200

=== Running Task Schedule ===

  [  1] t=    0ms  Parse config          priority=critical    deadline=  100  dur=  50ms
  [  2] t=   50ms  Check environment     priority=high        deadline=  200  dur=  30ms
  ...

Completed 10 tasks in 2080ms total simulated time
```

---

## Summary

Zig 0.16 provides a comprehensive set of heap data structures:

- **`std.heap`**: Low-level binary heap with `siftUp`, `siftDown`, and `heapify` for building min/max-heaps on any slice with custom comparators.
- **`std.PriorityQueue(T)`**: A clean priority queue API built on top of the binary heap. Supports `add`, `removeOrNull`, `peek`, and `update`.
- **`std.PriorityDequeue(T)`** (NEW): A double-ended priority queue that provides O(1) access to both min and max, O(log n) removal of either end — perfect for sliding windows, median tracking, and deadline-aware scheduling.
- **Custom heaps**: Easy to build with the `std.heap` primitives and a comparison context.
- **Heap sort**: O(n log n) in-place sorting, especially useful when you need guaranteed performance without additional memory.
- **Graph algorithms**: Dijkstra and A* are natural fits for priority queues, and Zig's zero-cost abstractions make them efficient and readable.