# JSON in Zig 0.16

A comprehensive guide to working with JSON data in Zig 0.16, covering parsing, serialization, dynamic JSON with `ObjectMap`, streaming with `DynamicStream`, and building a configuration file parser.

---

## Chapter 1: Introduction to std.json

Zig 0.16's `std.json` module provides a complete, zero-allocation-when-possible JSON toolkit. The major 0.16 changes include:

- **`ObjectMap`** replaces the previous flat array-of-key-value-pairs for JSON objects, providing O(log n) or O(1) lookup by key.
- **`DynamicStream`** enables true streaming JSON parsing for large files and network responses.
- **`bufPrintZ` → `bufPrintSentinel`**: Renamed for clarity.
- All parse functions accept a `std.json.ParseOptions` for fine-grained control.

### JSON Value Types in Zig

```zig
const std = @import("std");

// The std.json.Value union represents any JSON value:
// .null
// .bool    (bool)
// .integer (i64)
// .float   (f64)
// .number_string ([]const u8)  -- for numbers that don't fit in i64/f64
// .string  ([]const u8)
// .array   (std.json.Array)
// .object  (std.json.ObjectMap)
```

---

## Chapter 2: Parsing JSON — std.json.parse

`std.json.parse` is the primary way to parse JSON into Zig types. It uses compile-time type reflection to map JSON fields to struct fields automatically.

### Basic Struct Parsing

```zig
const std = @import("std");

const User = struct {
    name: []const u8,
    age: u32,
    email: ?[]const u8 = null, // optional field
    active: bool = true,       // default value
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\{
        \\    "name": "Alice Johnson",
        \\    "age": 30,
        \\    "email": "alice@example.com",
        \\    "active": true
        \\}
    ;

    const parsed = try std.json.parseFromSlice(User, allocator, input, .{});
    defer parsed.deinit();

    const user = parsed.value;
    try w.print("User: {s}\n", .{user.name});
    try w.print("Age:  {d}\n", .{user.age});
    if (user.email) |email| {
        try w.print("Email: {s}\n", .{email});
    }
    try w.print("Active: {}\n", .{user.active});
}
```

### Nested Structs

```zig
const std = @import("std");

const Address = struct {
    street: []const u8,
    city: []const u8,
    zipcode: []const u8,
};

const Person = struct {
    name: []const u8,
    age: u32,
    address: Address,
    hobbies: [][]const u8,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\{
        \\    "name": "Bob Smith",
        \\    "age": 28,
        \\    "address": {
        \\        "street": "123 Main St",
        \\        "city": "Springfield",
        \\        "zipcode": "62701"
        \\    },
        \\    "hobbies": ["reading", "gaming", "hiking"]
        \\}
    ;

    const parsed = try std.json.parseFromSlice(Person, allocator, input, .{});
    defer parsed.deinit();

    const person = parsed.value;
    try w.print("Name:    {s}\n", .{person.name});
    try w.print("Age:     {d}\n", .{person.age});
    try w.print("Address: {s}, {s} {s}\n", .{
        person.address.street,
        person.address.city,
        person.address.zipcode,
    });
    try w.print("Hobbies: {s}\n", .{person.hobbies});
}
```

### Parse Options

```zig
const parsed = try std.json.parseFromSlice(
    MyType,
    allocator,
    input,
    .{
        .ignore_unknown_fields = true, // Don't error on unknown JSON keys
        .allocate = .alloc_if_needed,  // Control allocation behavior
    },
);
```

---

## Chapter 3: std.json.parseFromSlice — Parse from a Byte Slice

`parseFromSlice` is the most common entry point. It accepts a `[]const u8` and returns a `Parsed(T)` struct that owns the allocated data.

### Parsing Arrays of Objects

```zig
const std = @import("std");

const Product = struct {
    id: u32,
    name: []const u8,
    price: f64,
    in_stock: bool,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\[
        \\    {"id": 1, "name": "Widget A", "price": 9.99, "in_stock": true},
        \\    {"id": 2, "name": "Gadget B", "price": 24.99, "in_stock": false},
        \\    {"id": 3, "name": "Tool C", "price": 49.99, "in_stock": true},
        \\    {"id": 4, "name": "Device D", "price": 14.50, "in_stock": true}
        \\]
    ;

    const parsed = try std.json.parseFromSlice([]Product, allocator, input, .{});
    defer parsed.deinit();

    var total_value: f64 = 0;
    var in_stock_count: u32 = 0;

    for (parsed.value) |product| {
        try w.print("  #{d}: {s:15s} ${d:6.2}  stock={}\n", .{
            product.id,
            product.name,
            product.price,
            product.in_stock,
        });
        total_value += product.price;
        if (product.in_stock) in_stock_count += 1;
    }

    try w.print("\n  Total products:  {d}\n", .{parsed.value.len});
    try w.print("  In stock:        {d}\n", .{in_stock_count});
    try w.print("  Total value:     ${d:.2}\n", .{total_value});
}
```

### Parsing Enums

```zig
const std = @import("std");

const Status = enum {
    pending,
    active,
    completed,
    failed,

    pub fn jsonStringify(self: Status, writer: anytype) !void {
        try writer.write(@tagName(self));
    }
};

const Job = struct {
    id: u32,
    status: Status,
    message: []const u8,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\{"id": 42, "status": "completed", "message": "Done!"}
    ;

    const parsed = try std.json.parseFromSlice(Job, allocator, input, .{});
    defer parsed.deinit();

    try w.print("Job #{d}: status={}, message={s}\n", .{
        parsed.value.id,
        parsed.value.status,
        parsed.value.message,
    });
}
```

---

## Chapter 4: std.json.parseFromTokenSlice — Streaming Parse

For advanced use cases, `parseFromTokenSlice` lets you parse from a pre-tokenized slice. This is useful when you want to inspect or filter tokens before parsing.

### Token-Based Parsing

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\{"name": "test", "value": 42, "tags": ["a", "b"]}
    ;

    // First, tokenize the input
    var scanner = std.json.Scanner.initCompleteInput(allocator, input);
    defer scanner.deinit();

    // Tokenize and count
    var token_count: usize = 0;
    while (true) {
        const token = scanner.next() catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        _ = token;
        token_count += 1;
    }

    try w.print("Token count: {d}\n", .{token_count});
    _ = allocator;
}
```

### Using Token Slice for Partial Parsing

```zig
const std = @import("std");

// Parse only the "name" field from a JSON object
fn extractName(allocator: std.mem.Allocator, input: []const u8) ![]const u8 {
    // Tokenize
    var scanner = std.json.Scanner.initCompleteInput(allocator, input);
    defer scanner.deinit();

    var token_buf = std.ArrayList(std.json.Token).init(allocator);
    defer token_buf.deinit();

    while (true) {
        const token = scanner.next() catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        try token_buf.append(token);
    }

    // Find "name" key and return its string value
    for (token_buf.items, 0..) |token, i| {
        if (token == .string) {
            if (std.mem.eql(u8, token.string, "name")) {
                // Next non-whitespace token should be the colon, then the value
                if (i + 2 < token_buf.items.len and token_buf.items[i + 2] == .string) {
                    return token_buf.items[i + 2].string;
                }
            }
        }
    }

    return error.NameNotFound;
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input = `{"name": "Alice", "age": 30, "city": "NYC"}`;
    const name = try extractName(allocator, input);
    try w.print("Extracted name: {s}\n", .{name});
}
```

---

## Chapter 5: std.json.stringify — Serializing to JSON

`std.json.stringify` converts Zig values to JSON. It writes directly to an `std.Io.Writer`.

### Basic Serialization

```zig
const std = @import("std");

const Config = struct {
    app_name: []const u8,
    version: []const u8,
    debug: bool,
    max_connections: u32,
    timeout_ms: u32,
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const config = Config{
        .app_name = "MyApp",
        .version = "1.0.0",
        .debug = false,
        .max_connections = 100,
        .timeout_ms = 5000,
    };

    try w.print("Config as JSON:\n", .{});
    std.json.stringify(config, .{ .whitespace = .indent_2 }, w);
    try w.print("\n", .{});
}
```

### Formatting Options

```zig
// Compact (no whitespace)
std.json.stringify(data, .{}, writer);

// Pretty-printed with 2-space indentation
std.json.stringify(data, .{ .whitespace = .indent_2 }, writer);

// Pretty-printed with 4-space indentation
std.json.stringify(data, .{ .whitespace = .indent_4 }, writer);

// Single space after separators
std.json.stringify(data, .{ .whitespace = .space_after_sep }, writer);
```

### Custom Serialization with jsonStringify

Add a `jsonStringify` method to control how your type serializes:

```zig
const std = @import("std");

const Timestamp = struct {
    seconds: i64,
    nanos: u32,

    pub fn jsonStringify(self: Timestamp, writer: anytype) !void {
        // Serialize as ISO 8601 string
        try writer.write("\"");
        const epoch = std.time.epoch.EpochSeconds{ .secs = @intCast(self.seconds) };
        const day = epoch.getDay();
        const year = day.getYear();
        const month = day.getMonth();
        const day_num = day.getDay();
        try std.fmt.format(writer, "{d:04}-{d:02}-{d:02}T00:00:00Z", .{
            year, @intFromEnum(month), day_num,
        });
        try writer.write("\"");
    }
};

const Event = struct {
    name: []const u8,
    timestamp: Timestamp,
    count: u32,
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const event = Event{
        .name = "user_signup",
        .timestamp = .{ .seconds = 1700000000, .nanos = 0 },
        .count = 42,
    };

    std.json.stringify(event, .{ .whitespace = .indent_2 }, w);
    try w.print("\n", .{});
}
```

---

## Chapter 6: std.json.stringifyAlloc — Allocate and Serialize

When you need a `[]const u8` instead of writing to a writer, use `stringifyAlloc`:

```zig
const std = @import("std");

const ServerConfig = struct {
    host: []const u8 = "0.0.0.0",
    port: u16 = 8080,
    workers: u32 = 4,
    tls: bool = false,
    allowed_origins: [][]const u8 = &.{},
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const config = ServerConfig{
        .host = "127.0.0.1",
        .port = 3000,
        .workers = 8,
        .tls = true,
        .allowed_origins = &.{
            "https://example.com",
            "https://app.example.com",
        },
    };

    const json_string = try std.json.stringifyAlloc(allocator, config, .{
        .whitespace = .indent_2,
    });
    defer allocator.free(json_string);

    try w.print("Allocated JSON ({d} bytes):\n{s}\n", .{
        json_string.len,
        json_string,
    });
}
```

### Serializing to a File

```zig
fn writeJsonToFile(
    allocator: std.mem.Allocator,
    path: []const u8,
    data: anytype,
) !void {
    const file = try std.fs.cwd().createFile(path, .{});
    defer file.close();

    const writer = file.writer();
    std.json.stringify(data, .{ .whitespace = .indent_2 }, writer);
    try writer.writeAll("\n");
}
```

---

## Chapter 7: Dynamic JSON with std.json.Value (ObjectMap, Array)

When you don't know the JSON structure at compile time, use `std.json.Value`. In Zig 0.16, JSON objects use `ObjectMap` for efficient key-based access.

### Parsing Dynamic JSON

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\{
        \\    "name": "Zig Project",
        \\    "version": "0.16.0",
        \\    "dependencies": {
        \\        "zlib": "1.3.1",
        \\        "libcurl": "8.5.0"
        \\    },
        \\    "features": ["fast", "safe", "composable"],
        \\    "metadata": null,
        \\    "stats": {
        \\        "stars": 25000,
        \\        "forks": 1800,
        \\        "is_public": true
        \\    }
        \\}
    ;

    const parsed = try std.json.parseFromSlice(
        std.json.Value,
        allocator,
        input,
        .{ .allocate = .alloc_always },
    );
    defer parsed.deinit();

    const root = parsed.value;

    // Access string fields
    if (root.object.get("name")) |name_val| {
        try w.print("Name: {s}\n", .{name_val.string});
    }

    // Access nested object using ObjectMap
    if (root.object.get("stats")) |stats| {
        if (stats.object.get("stars")) |stars| {
            try w.print("Stars: {d}\n", .{stars.integer});
        }
        if (stats.object.get("is_public")) |pub_val| {
            try w.print("Public: {}\n", .{pub_val.bool});
        }
    }

    // Iterate over array
    if (root.object.get("features")) |features| {
        try w.print("Features: ", .{});
        for (features.array.items) |item| {
            try w.print("{s} ", .{item.string});
        }
        try w.print("\n");
    }

    // Iterate over object keys
    if (root.object.get("dependencies")) |deps| {
        try w.print("Dependencies:\n", .{});
        var it = deps.object.iterator();
        while (it.next()) |entry| {
            try w.print("  {s}: {s}\n", .{
                entry.key_ptr.*,
                entry.value_ptr.*.string,
            });
        }
    }
}
```

### Building Dynamic JSON

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Build a dynamic JSON object using ObjectMap
    var root = std.json.ObjectMap.init(allocator);
    defer root.deinit();

    try root.put("name", .{ .string = "Dynamic Example" });
    try root.put("version", .{ .string = "1.0.0" });
    try root.put("count", .{ .integer = 42 });
    try root.put("ratio", .{ .float = 3.14 });
    try root.put("active", .{ .bool = true });
    try root.put("nothing", .null);

    // Add an array
    var tags = std.json.Array.init(allocator);
    defer tags.deinit();

    try tags.append(.{ .string = "zig" });
    try tags.append(.{ .string = "json" });
    try tags.append(.{ .string = "dynamic" });

    try root.put("tags", .{ .array = tags });

    // Add a nested object
    var nested = std.json.ObjectMap.init(allocator);
    defer nested.deinit();

    try nested.put("width", .{ .integer = 1920 });
    try nested.put("height", .{ .integer = 1080 });
    try nested.put("fullscreen", .{ .bool = false });

    try root.put("display", .{ .object = nested });

    // Serialize
    const value = std.json.Value{ .object = root };
    std.json.stringify(value, .{ .whitespace = .indent_2 }, w);
    try w.print("\n", .{});
}
```

### ObjectMap Operations

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    var obj = std.json.ObjectMap.init(allocator);
    defer obj.deinit();

    // Insert
    try obj.put("key1", .{ .string = "value1" });
    try obj.put("key2", .{ .integer = 100 });
    try obj.put("key3", .{ .bool = true });

    // Get
    if (obj.get("key1")) |val| {
        try w.print("key1 = {s}\n", .{val.string});
    }

    // Check existence
    try w.print("has key2: {}\n", .{obj.contains("key2")});
    try w.print("has missing: {}\n", .{obj.contains("missing")});

    // Count
    try w.print("count: {d}\n", .{obj.count()});

    // Delete
    _ = obj.swapRemove("key2");
    try w.print("After remove, count: {d}\n", .{obj.count()});

    // Iterate
    try w.print("Remaining keys: ", .{});
    var it = obj.iterator();
    while (it.next()) |entry| {
        try w.print("{s} ", .{entry.key_ptr.*});
    }
    try w.print("\n");
}
```

---

## Chapter 8: JSON Streaming with std.json.DynamicStream

Zig 0.16 introduces `std.json.DynamicStream` for true streaming JSON parsing. This allows you to process very large JSON files without loading the entire document into memory.

### Basic Streaming

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\[
        \\    {"id": 1, "name": "Alice", "score": 95},
        \\    {"id": 2, "name": "Bob", "score": 87},
        \\    {"id": 3, "name": "Charlie", "score": 92},
        \\    {"id": 4, "name": "Diana", "score": 98}
        \\]
    ;

    var stream = std.json.DynamicStream.init(allocator, input);
    defer stream.deinit();

    try w.print("Streaming JSON array items:\n", .{});

    while (try stream.nextValue(allocator)) |value| {
        defer value.deinit(allocator);

        if (value != .object) continue;

        const id = value.object.get("id") orelse continue;
        const name = value.object.get("name") orelse continue;
        const score = value.object.get("score") orelse continue;

        try w.print("  #{d}: {s:10s} score={d}\n", .{
            id.integer,
            name.string,
            score.integer,
        });
    }
}
```

### Streaming Large Files

```zig
const std = @import("std");

const BUFFER_SIZE = 64 * 1024;

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const file = try std.fs.cwd().openFile("large-data.json", .{});
    defer file.close();

    var buf: [BUFFER_SIZE]u8 = undefined;
    var stream = std.json.DynamicStream.init(allocator, &buf);
    defer stream.deinit();

    var count: u64 = 0;
    var total_size: u64 = 0;

    while (true) {
        const bytes_read = file.read(&buf) catch |err| switch (err) {
            error.EndOfStream => break,
            else => return err,
        };
        if (bytes_read == 0) break;

        var chunk_stream = std.json.DynamicStream.init(allocator, buf[0..bytes_read]);
        defer chunk_stream.deinit();

        while (try chunk_stream.nextValue(allocator)) |value| {
            defer value.deinit(allocator);
            count += 1;
            total_size += bytes_read;
        }
    }

    try w.print("Processed {d} JSON values\n", .{count});
    try w.print("Total bytes read: {d}\n", .{total_size});
}
```

### Streaming with Event-Based Processing

```zig
const std = @import("std");

const JsonEvent = enum {
    object_begin,
    object_end,
    array_begin,
    array_end,
    string,
    number,
    @"true",
    @"false",
    null,
    key,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    const input =
        \\{"users": [{"name": "Alice", "admin": true}, {"name": "Bob", "admin": false}]}
    ;

    var stream = std.json.DynamicStream.init(allocator, input);
    defer stream.deinit();

    var depth: u32 = 0;
    var in_users: bool = false;

    while (try stream.nextValue(allocator)) |value| {
        defer value.deinit(allocator);

        switch (value) {
            .object_begin => {
                const indent = "  " ** depth;
                try w.print("{s}{{\n", .{indent});
                depth += 1;
            },
            .object_end => {
                depth -= 1;
                const indent = "  " ** depth;
                try w.print("{s}}}\n", .{indent});
            },
            .array_begin => {
                try w.print("[\n", .{});
                depth += 1;
            },
            .array_end => {
                depth -= 1;
                try w.print("]\n", .{});
            },
            .string => |s| {
                const indent = "  " ** depth;
                try w.print("{s}{s}\n", .{ indent, s });
            },
            .integer => |i| {
                const indent = "  " ** depth;
                try w.print("{s}{}\n", .{ indent, i });
            },
            .bool => |b| {
                const indent = "  " ** depth;
                try w.print("{s}{}\n", .{ indent, b });
            },
            .null => {
                const indent = "  " ** depth;
                try w.print("{s}null\n", .{indent});
            },
            else => {},
        }
    }

    _ = in_users;
}
```

---

## Chapter 9: Working with JSON in HTTP APIs

A common use case for JSON is communicating with REST APIs. Here's how to handle JSON request/response cycles.

### Building an API Request Body

```zig
const std = @import("std");

const CreatePostRequest = struct {
    title: []const u8,
    body: []const u8,
    user_id: u32,
};

const ApiResponse = struct {
    id: u32,
    title: []const u8,
    body: []const u8,
    user_id: u32,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Build request
    const request = CreatePostRequest{
        .title = "Hello Zig 0.16",
        .body = "JSON handling in Zig is excellent!",
        .user_id = 1,
    };

    // Serialize to JSON string
    const body = try std.json.stringifyAlloc(allocator, request, .{
        .whitespace = .indent_2,
    });
    defer allocator.free(body);

    try w.print("Request body:\n{s}\n", .{body});

    // Simulate receiving a response
    const response_json =
        \\{
        \\    "id": 101,
        \\    "title": "Hello Zig 0.16",
        \\    "body": "JSON handling in Zig is excellent!",
        \\    "user_id": 1
        \\}
    ;

    const parsed = try std.json.parseFromSlice(
        ApiResponse,
        allocator,
        response_json,
        .{},
    );
    defer parsed.deinit();

    const response = parsed.value;
    try w.print("\nResponse: post #{d} created\n", .{response.id});
    try w.print("  Title: {s}\n", .{response.title});
}
```

### Handling Optional and Nullable Fields

```zig
const std = @import("std");

const UserProfile = struct {
    id: u32,
    username: []const u8,
    email: ?[]const u8,
    bio: ?[]const u8 = null,
    avatar_url: ?[]const u8 = null,
    followers: u32 = 0,
    is_verified: bool = false,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Full profile
    const full_json =
        \\{
        \\    "id": 1,
        \\    "username": "alice",
        \\    "email": "alice@example.com",
        \\    "bio": "Zig enthusiast",
        \\    "avatar_url": "https://example.com/avatar.png",
        \\    "followers": 1500,
        \\    "is_verified": true
        \\}
    ;

    const full = try std.json.parseFromSlice(UserProfile, allocator, full_json, .{});
    defer full.deinit();

    try w.print("Full profile: {s} ({d} followers, verified={})\n", .{
        full.value.username,
        full.value.followers,
        full.value.is_verified,
    });

    // Minimal profile (missing optional fields)
    const minimal_json =
        \\{"id": 2, "username": "bob", "email": "bob@example.com"}
    ;

    const minimal = try std.json.parseFromSlice(UserProfile, allocator, minimal_json, .{});
    defer minimal.deinit();

    try w.print("Minimal profile: {s} (followers={d}, verified={})\n", .{
        minimal.value.username,
        minimal.value.followers,
        minimal.value.is_verified,
    });
}
```

### JSON Patch (Partial Updates)

```zig
const std = @import("std");

const ServerConfig = struct {
    host: []const u8,
    port: u16,
    debug: bool,
    log_level: []const u8,
};

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Current config
    const current = ServerConfig{
        .host = "0.0.0.0",
        .port = 8080,
        .debug = true,
        .log_level = "debug",
    };

    // Parse a partial update (only fields that changed)
    const patch_json =
        \\{"port": 3000, "debug": false, "log_level": "info"}
    ;

    // Parse the patch into dynamic JSON
    const patch_parsed = try std.json.parseFromSlice(
        std.json.Value,
        allocator,
        patch_json,
        .{ .allocate = .alloc_always },
    );
    defer patch_parsed.deinit();

    const patch = patch_parsed.value.object;

    // Apply patch fields to create updated config
    const updated = ServerConfig{
        .host = if (patch.get("host")) |v| v.string else current.host,
        .port = if (patch.get("port")) |v| @intCast(v.integer) else current.port,
        .debug = if (patch.get("debug")) |v| v.bool else current.debug,
        .log_level = if (patch.get("log_level")) |v| v.string else current.log_level,
    };

    try w.print("Updated config:\n", .{});
    std.json.stringify(updated, .{ .whitespace = .indent_2 }, w);
    try w.print("\n", .{});
}
```

---

## Chapter 10: JSON Schema Validation Patterns

Zig doesn't ship a built-in JSON Schema validator, but you can implement validation patterns using Zig's type system and runtime checks.

### Validation with Result Types

```zig
const std = @import("std");

const ValidationError = struct {
    field: []const u8,
    message: []const u8,
};

const ValidationResult = struct {
    errors: []ValidationError,
    allocator: std.mem.Allocator,

    pub fn init(allocator: std.mem.Allocator) ValidationResult {
        return .{ .errors = &.{}, .allocator = allocator };
    }

    pub fn addError(self: *ValidationResult, field: []const u8, message: []const u8) !void {
        const err_list = self.allocator.alloc(ValidationError, self.errors.len + 1) catch return;
        @memcpy(err_list[0..self.errors.len], self.errors);
        err_list[self.errors.len] = .{ .field = field, .message = message };
        self.errors = err_list;
    }

    pub fn isValid(self: *const ValidationResult) bool {
        return self.errors.len == 0;
    }

    pub fn deinit(self: *ValidationResult) void {
        if (self.errors.len > 0) {
            self.allocator.free(self.errors);
        }
    }
};

fn validateRequired(value: ?std.json.Value, field: []const u8, result: *ValidationResult) !void {
    if (value == null) {
        try result.addError(field, "field is required");
    }
}

fn validateString(
    value: ?std.json.Value,
    field: []const u8,
    result: *ValidationResult,
    min_len: ?usize,
    max_len: ?usize,
    pattern: ?[]const u8,
) !void {
    if (value == null) return;
    const str = value.?.string;

    if (min_len) |min| {
        if (str.len < min) {
            try result.addError(field, "too short");
        }
    }
    if (max_len) |max| {
        if (str.len > max) {
            try result.addError(field, "too long");
        }
    }
    if (pattern) |pat| {
        // Simple pattern matching (for full regex, use a regex library)
        if (!containsPattern(str, pat)) {
            try result.addError(field, "does not match pattern");
        }
    }
}

fn validateNumber(
    value: ?std.json.Value,
    field: []const u8,
    result: *ValidationResult,
    min: ?i64,
    max: ?i64,
) !void {
    if (value == null) return;
    const num = value.?.integer;

    if (min) |m| {
        if (num < m) {
            try result.addError(field, "below minimum");
        }
    }
    if (max) |m| {
        if (num > m) {
            try result.addError(field, "above maximum");
        }
    }
}

fn containsPattern(haystack: []const u8, needle: []const u8) bool {
    return std.mem.indexOf(u8, haystack, needle) != null;
}
```

### Full Validation Example

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();

    // Valid config
    const valid_input =
        \\{
        \\    "app_name": "MyApp",
        \\    "version": "1.0.0",
        \\    "port": 3000,
        \\    "debug": false,
        \\    "database": {
        \\        "host": "localhost",
        \\        "port": 5432,
        \\        "name": "mydb"
        \\    }
        \\}
    ;

    const parsed = try std.json.parseFromSlice(
        std.json.Value,
        allocator,
        valid_input,
        .{ .allocate = .alloc_always },
    );
    defer parsed.deinit();

    const obj = parsed.value.object;
    var result = ValidationResult.init(allocator);
    defer result.deinit();

    // Validate required top-level fields
    try validateRequired(obj.get("app_name"), "app_name", &result);
    try validateRequired(obj.get("version"), "version", &result);
    try validateRequired(obj.get("port"), "port", &result);

    // Validate string constraints
    try validateString(obj.get("app_name"), "app_name", &result, 1, 50, null);
    try validateString(obj.get("version"), "version", &result, 5, 10, null);

    // Validate number constraints
    try validateNumber(obj.get("port"), "port", &result, 1024, 65535);

    // Validate nested object
    if (obj.get("database")) |db_val| {
        const db = db_val.object;
        try validateRequired(db.get("host"), "database.host", &result);
        try validateRequired(db.get("port"), "database.port", &result);
        try validateRequired(db.get("name"), "database.name", &result);
        try validateNumber(db.get("port"), "database.port", &result, 1, 65535);
    }

    if (result.isValid()) {
        try w.print("✓ Configuration is valid!\n", .{});
    } else {
        try w.print("✗ Validation errors ({d}):\n", .{result.errors.len});
        for (result.errors) |err| {
            try w.print("  - {s}: {s}\n", .{ err.field, err.message });
        }
    }
}
```

---

## Chapter 11: Project — A JSON Configuration File Parser & Validator

This project builds a full-featured configuration file parser that reads, validates, and manages JSON configuration files with defaults, environment variable overrides, and schema validation.

### Project Structure

```
config-parser/
├── build.zig
├── src/
│   └── main.zig
└── example.json
```

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "config-parser",
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

    const run_step = b.step("run", "Run the config parser");
    run_step.dependOn(&run_cmd.step);
}
```

### example.json

```json
{
    "app": {
        "name": "MyApp",
        "version": "1.2.0",
        "environment": "production"
    },
    "server": {
        "host": "0.0.0.0",
        "port": 8080,
        "workers": 4,
        "tls": true,
        "tls_cert": "/etc/ssl/cert.pem"
    },
    "database": {
        "host": "localhost",
        "port": 5432,
        "name": "myapp_prod",
        "pool_size": 20,
        "timeout_ms": 5000
    },
    "logging": {
        "level": "info",
        "format": "json",
        "output": "/var/log/myapp/app.log"
    },
    "features": {
        "rate_limit": true,
        "cache_enabled": true,
        "max_request_size_mb": 50,
        "allowed_origins": [
            "https://myapp.com",
            "https://app.myapp.com"
        ]
    }
}
```

### src/main.zig

```zig
const std = @import("std");

// ============================================================
// Configuration Types
// ============================================================

const AppConfig = struct {
    app: AppSection,
    server: ServerSection,
    database: DatabaseSection,
    logging: LoggingSection,
    features: FeaturesSection,
};

const AppSection = struct {
    name: []const u8,
    version: []const u8,
    environment: []const u8,
};

const ServerSection = struct {
    host: []const u8,
    port: u16,
    workers: u32,
    tls: bool,
    tls_cert: ?[]const u8 = null,
};

const DatabaseSection = struct {
    host: []const u8,
    port: u16,
    name: []const u8,
    pool_size: u32 = 10,
    timeout_ms: u32 = 3000,
};

const LoggingSection = struct {
    level: []const u8 = "info",
    format: []const u8 = "text",
    output: []const u8 = "stdout",
};

const FeaturesSection = struct {
    rate_limit: bool = true,
    cache_enabled: bool = false,
    max_request_size_mb: u32 = 10,
    allowed_origins: [][]const u8 = &.{},
};

// ============================================================
// Validation
// ============================================================

const ValidationError = struct {
    path: []const u8,
    message: []const u8,
};

const Validator = struct {
    errors: std.ArrayList(ValidationError),
    allocator: std.mem.Allocator,

    fn init(allocator: std.mem.Allocator) Validator {
        return .{
            .errors = std.ArrayList(ValidationError).init(allocator),
            .allocator = allocator,
        };
    }

    fn deinit(self: *Validator) void {
        self.errors.deinit();
    }

    fn addError(self: *Validator, path: []const u8, msg: []const u8) !void {
        const path_copy = try self.allocator.dupe(u8, path);
        const msg_copy = try self.allocator.dupe(u8, msg);
        try self.errors.append(.{ .path = path_copy, .message = msg_copy });
    }

    fn isValid(self: *const Validator) bool {
        return self.errors.items.len == 0;
    }

    fn checkRange(self: *Validator, path: []const u8, value: u64, min: u64, max: u64) void {
        if (value < min) {
            self.addError(path, "value below minimum") catch {};
        } else if (value > max) {
            self.addError(path, "value above maximum") catch {};
        }
    }

    fn checkOneOf(self: *Validator, path: []const u8, value: []const u8, options: []const []const u8) void {
        for (options) |opt| {
            if (std.mem.eql(u8, value, opt)) return;
        }
        self.addError(path, "invalid option value") catch {};
    }

    fn checkNotEmpty(self: *Validator, path: []const u8, value: []const u8) void {
        if (value.len == 0) {
            self.addError(path, "must not be empty") catch {};
        }
    }

    fn checkLength(self: *Validator, path: []const u8, value: []const u8, min: usize, max: usize) void {
        if (value.len < min) {
            self.addError(path, "too short") catch {};
        } else if (value.len > max) {
            self.addError(path, "too long") catch {};
        }
    }

    fn checkTlsCert(self: *Validator, server: *const ServerSection) void {
        if (server.tls and server.tls_cert == null) {
            self.addError("server.tls_cert", "required when TLS is enabled") catch {};
        }
    }
};

fn validateConfig(allocator: std.mem.Allocator, config: *const AppConfig) !Validator {
    var v = Validator.init(allocator);
    defer v.deinit();

    // App section
    v.checkNotEmpty("app.name", config.app.name);
    v.checkLength("app.name", config.app.name, 1, 100);
    v.checkLength("app.version", config.app.version, 1, 20);
    v.checkOneOf("app.environment", config.app.environment, &.{
        "development", "staging", "production", "test",
    });

    // Server section
    v.checkRange("server.port", config.server.port, 1, 65535);
    v.checkRange("server.workers", config.server.workers, 1, 1024);
    v.checkTlsCert(&config.server);

    // Database section
    v.checkNotEmpty("database.host", config.database.host);
    v.checkNotEmpty("database.name", config.database.name);
    v.checkRange("database.port", config.database.port, 1, 65535);
    v.checkRange("database.pool_size", config.database.pool_size, 1, 1000);
    v.checkRange("database.timeout_ms", config.database.timeout_ms, 100, 60000);

    // Logging section
    v.checkOneOf("logging.level", config.logging.level, &.{
        "trace", "debug", "info", "warn", "error", "fatal",
    });
    v.checkOneOf("logging.format", config.logging.format, &.{
        "text", "json",
    });

    // Features section
    v.checkRange("features.max_request_size_mb", config.features.max_request_size_mb, 1, 1024);
    if (config.features.allowed_origins.len == 0 and config.server.tls) {
        try v.addError("features.allowed_origins", "should have at least one origin when TLS is enabled");
    }

    // Clone errors to return
    var result = Validator.init(allocator);
    for (v.errors.items) |err| {
        try result.errors.append(err);
    }

    return result;
}

// ============================================================
// Config Loading and Merging
// ============================================================

fn loadConfig(allocator: std.mem.Allocator, path: []const u8) !AppConfig {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    const stat = try file.stat();
    const content = try allocator.alloc(u8, stat.size);
    defer allocator.free(content);

    _ = try file.readAll(content);

    const parsed = try std.json.parseFromSlice(
        AppConfig,
        allocator,
        content,
        .{ .ignore_unknown_fields = true },
    );
    defer parsed.deinit();

    return parsed.value;
}

fn printConfig(config: *const AppConfig, writer: anytype) !void {
    try writer.print("=== Application Configuration ===\n\n", .{});

    try writer.print("Application:\n", .{});
    try writer.print("  Name:         {s}\n", .{config.app.name});
    try writer.print("  Version:      {s}\n", .{config.app.version});
    try writer.print("  Environment:  {s}\n", .{config.app.environment});

    try writer.print("\nServer:\n", .{});
    try writer.print("  Host:         {s}\n", .{config.server.host});
    try writer.print("  Port:         {d}\n", .{config.server.port});
    try writer.print("  Workers:      {d}\n", .{config.server.workers});
    try writer.print("  TLS:          {}\n", .{config.server.tls});
    if (config.server.tls_cert) |cert| {
        try writer.print("  TLS Cert:     {s}\n", .{cert});
    }

    try writer.print("\nDatabase:\n", .{});
    try writer.print("  Host:         {s}\n", .{config.database.host});
    try writer.print("  Port:         {d}\n", .{config.database.port});
    try writer.print("  Name:         {s}\n", .{config.database.name});
    try writer.print("  Pool Size:    {d}\n", .{config.database.pool_size});
    try writer.print("  Timeout:      {d}ms\n", .{config.database.timeout_ms});

    try writer.print("\nLogging:\n", .{});
    try writer.print("  Level:        {s}\n", .{config.logging.level});
    try writer.print("  Format:       {s}\n", .{config.logging.format});
    try writer.print("  Output:       {s}\n", .{config.logging.output});

    try writer.print("\nFeatures:\n", .{});
    try writer.print("  Rate Limit:   {}\n", .{config.features.rate_limit});
    try writer.print("  Cache:        {}\n", .{config.features.cache_enabled});
    try writer.print("  Max Req Size: {d}MB\n", .{config.features.max_request_size_mb});
    if (config.features.allowed_origins.len > 0) {
        try writer.print("  Origins:      ", .{});
        for (config.features.allowed_origins) |origin| {
            try writer.print("{s} ", .{origin});
        }
        try writer.print("\n", .{});
    }
}

fn printValidationErrors(errors: []ValidationError, writer: anytype) !void {
    try writer.print("Validation errors ({d}):\n", .{errors.len});
    for (errors) |err| {
        try writer.print("  ✗ {s}: {s}\n", .{ err.path, err.message });
    }
}

fn exportConfig(allocator: std.mem.Allocator, config: *const AppConfig, path: []const u8) !void {
    const file = try std.fs.cwd().createFile(path, .{});
    defer file.close();

    const writer = file.writer();
    try writer.writeAll("// Auto-generated configuration file\n");
    std.json.stringify(config, .{ .whitespace = .indent_2 }, writer);
    try writer.writeAll("\n");
}

fn generateDefaultConfig(allocator: std.mem.Allocator, writer: anytype) !void {
    const defaults = AppConfig{
        .app = .{
            .name = "MyApp",
            .version = "0.1.0",
            .environment = "development",
        },
        .server = .{
            .host = "127.0.0.1",
            .port = 3000,
            .workers = 2,
            .tls = false,
            .tls_cert = null,
        },
        .database = .{
            .host = "localhost",
            .port = 5432,
            .name = "myapp_dev",
            .pool_size = 5,
            .timeout_ms = 3000,
        },
        .logging = .{
            .level = "debug",
            .format = "text",
            .output = "stdout",
        },
        .features = .{
            .rate_limit = false,
            .cache_enabled = false,
            .max_request_size_mb = 10,
            .allowed_origins = &.{},
        },
    };

    try writer.print("Default configuration:\n\n", .{});
    std.json.stringify(defaults, .{ .whitespace = .indent_2 }, writer);
    try writer.print("\n\n", .{});
    _ = allocator;
}

// ============================================================
// Main
// ============================================================

fn printUsage(program: []const u8, writer: anytype) !void {
    try writer.print(
        \\JSON Configuration Parser & Validator for Zig 0.16
        \\
        \\Usage: {s} <command> [config.json]
        \\
        \\Commands:
        \\  load <path>     Load and display a config file
        \\  validate <path> Validate a config file
        \\  export <in> <out>  Load, validate, and export config
        \\  defaults        Show default configuration
        \\  help            Show this help message
        \\
        \\Examples:
        \\  {s} load example.json
        \\  {s} validate example.json
        \\  {s} export config.json config.validated.json
        \\  {s} defaults
        \\
    , .{ program, program, program, program, program });
}

pub fn main(init: std.process.Init) !void {
    const allocator = init.allocator;
    const stdout = init.io.stdout();
    const w = stdout.writer();
    const args = init.process.args;

    if (args.len < 2) {
        try printUsage(args[0], w);
        return;
    }

    const command = args[1];

    if (std.mem.eql(u8, command, "load")) {
        if (args.len < 3) {
            try w.print("Error: specify a config file path\n", .{});
            return;
        }
        const path = args[2];
        const config = loadConfig(allocator, path) catch |err| {
            try w.print("Error loading config: {}\n", .{err});
            return;
        };
        try printConfig(&config, w);

    } else if (std.mem.eql(u8, command, "validate")) {
        if (args.len < 3) {
            try w.print("Error: specify a config file path\n", .{});
            return;
        }
        const path = args[2];
        const config = loadConfig(allocator, path) catch |err| {
            try w.print("Error loading config: {}\n", .{err});
            return;
        };

        var validator = validateConfig(allocator, &config) catch |err| {
            try w.print("Error validating: {}\n", .{err});
            return;
        };
        defer {
            for (validator.errors.items) |err| {
                allocator.free(err.path);
                allocator.free(err.message);
            }
            validator.deinit();
        }

        if (validator.isValid()) {
            try w.print("✓ Configuration '{s}' is valid!\n", .{path});
        } else {
            try printValidationErrors(validator.errors.items, w);
        }

    } else if (std.mem.eql(u8, command, "export")) {
        if (args.len < 4) {
            try w.print("Error: specify input and output paths\n", .{});
            return;
        }
        const in_path = args[2];
        const out_path = args[3];

        const config = try loadConfig(allocator, in_path);
        var validator = try validateConfig(allocator, &config);
        defer {
            for (validator.errors.items) |err| {
                allocator.free(err.path);
                allocator.free(err.message);
            }
            validator.deinit();
        }

        if (!validator.isValid()) {
            try w.print("Cannot export: config has validation errors\n", .{});
            try printValidationErrors(validator.errors.items, w);
            return;
        }

        try exportConfig(allocator, &config, out_path);
        try w.print("✓ Exported validated config to '{s}'\n", .{out_path});

    } else if (std.mem.eql(u8, command, "defaults")) {
        try generateDefaultConfig(allocator, w);

    } else if (std.mem.eql(u8, command, "help") or std.mem.eql(u8, command, "--help") or std.mem.eql(u8, command, "-h")) {
        try printUsage(args[0], w);

    } else {
        try w.print("Unknown command: {s}\n\n", .{command});
        try printUsage(args[0], w);
    }
}
```

### Usage

```bash
# Load and display a config file
zig build run -- load example.json

# Validate a config file
zig build run -- validate example.json

# Export validated config
zig build run -- export example.json output.json

# Show default configuration
zig build run -- defaults

# Build optimized
zig build -Doptimize=ReleaseFast
./zig-out/bin/config-parser validate example.json
```

### Expected Output (validate)

```
✓ Configuration 'example.json' is valid!
```

### Expected Output (load)

```
=== Application Configuration ===

Application:
  Name:         MyApp
  Version:      1.2.0
  Environment:  production

Server:
  Host:         0.0.0.0
  Port:         8080
  Workers:      4
  TLS:          true
  TLS Cert:     /etc/ssl/cert.pem

Database:
  Host:         localhost
  Port:         5432
  Name:         myapp_prod
  Pool Size:    20
  Timeout:      5000ms

Logging:
  Level:        info
  Format:       json
  Output:       /var/log/myapp/app.log

Features:
  Rate Limit:   true
  Cache:        true
  Max Req Size: 50MB
  Origins:      https://myapp.com https://app.myapp.com
```

---

## Summary

Zig 0.16's `std.json` module is a complete JSON toolkit:

- **`std.json.parseFromSlice(T)`**: Parse JSON into typed Zig structs with automatic field mapping, optional fields, defaults, and nested types.
- **`std.json.parseFromTokenSlice`**: Advanced token-level parsing for partial or pre-processed JSON.
- **`std.json.stringify` / `stringifyAlloc`**: Serialize Zig types to JSON with formatting options (compact, indented).
- **`std.json.Value` with `ObjectMap`**: Dynamic JSON for when the schema is unknown at compile time. `ObjectMap` provides efficient key-based access.
- **`std.json.DynamicStream`**: True streaming JSON parsing for large files and network data.
- **Custom `jsonStringify`**: Control serialization format for any type.
- **`ParseOptions`**: Fine-grained control over parsing behavior including `ignore_unknown_fields`.
- **Validation patterns**: Combine typed parsing with runtime checks for robust configuration handling.

The 0.16 updates to `ObjectMap` and `DynamicStream` make Zig's JSON handling significantly more practical for production use, especially when dealing with large, dynamic, or partially-known JSON structures.