# Operating System Interfaces in Zig 0.16

A comprehensive guide to cross-platform OS abstractions, file systems, environment variables, process management, and compile-time target detection in Zig 0.16.

---

## Chapter 1: Introduction to std.os

Zig's standard library provides rich operating system interfaces through several key modules. In Zig 0.16, the landscape has been reorganized:

- **`std.os`** — Minimal namespace; most functions moved to `std.Io` or `std.posix.system`.
- **`std.posix`** — POSIX types, constants, and low-level syscalls.
- **`std.Io`** — High-level I/O: files, directories, networking, child processes.
- **`std.process`** — Process information, arguments, environment variables (now non-global).
- **`std.time`** — Timestamps, durations, sleeping.
- **`std.fs`** — File system operations via `std.Io`.
- **`std.Target`** — Compile-time target detection (CPU, OS, ABI).
- **`std.builtin`** — Compile-time builtins and target information.

The Zig philosophy: expose the raw OS interface faithfully, then layer safe abstractions on top.

---

## Chapter 2: Cross-Platform Abstractions in Zig

Zig achieves cross-platform code through several mechanisms:

### Comptime OS Detection

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const target = comptime std.Target.current;

    try writer.print("OS:   {s}\n", .{@tagName(target.os.tag)});
    try writer.print("CPU:  {s}\n", .{@tagName(target.cpu.arch)});
    try writer.print("ABI:  {s}\n", .{@tagName(target.abi)});
    try writer.print("Endian: {s}\n", .{@tagName(target.cpu.arch.endian())});

    // Comptime branching
    switch (target.os.tag) {
        .linux => try writer.print("Linux-specific code path\n", .{}),
        .macos => try writer.print("macOS-specific code path\n", .{}),
        .windows => try writer.print("Windows-specific code path\n", .{}),
        .wasi => try writer.print("WASI-specific code path\n", .{}),
        else => try writer.print("Unknown OS: {s}\n", .{@tagName(target.os.tag)}),
    }
}
```

### Conditional Compilation with `builtin`

```zig
const builtin = @import("builtin");

comptime {
    if (builtin.os.tag == .linux) {
        // Linux-specific comptime logic
    } else if (builtin.os.tag == .windows) {
        // Windows-specific comptime logic
    }
}
```

---

## Chapter 3: Environment Variables (Non-Global in 0.16)

In Zig 0.16, environment variables are **non-global**. They are accessed through the `std.process` namespace, specifically through the `Init` struct passed to `main`. This prevents hidden global state and makes testing easier.

### Reading Environment Variables

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Get environment map from the process
    const env_map = try init.process.getEnvMap(init.allocator);
    defer env_map.deinit(init.allocator);

    // List all environment variables
    try writer.print("Environment variables:\n", .{});
    var it = env_map.iterator();
    while (it.next()) |entry| {
        try writer.print("  {s} = {s}\n", .{ entry.key_ptr.*, entry.value_ptr.* });
    }

    // Get a specific variable
    if (env_map.get("HOME")) |home| {
        try writer.print("\nHOME = {s}\n", .{home});
    } else {
        try writer.print("\nHOME not set\n", .{});
    }

    if (env_map.get("PATH")) |path| {
        try writer.print("PATH = {s}\n", .{path});
    }

    if (env_map.get("USER")) |user| {
        try writer.print("USER = {s}\n", .{user});
    }
}
```

### Getting a Single Environment Variable

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Get a single environment variable
    if (init.process.getEnv("SHELL")) |shell| {
        try writer.print("Your shell: {s}\n", .{shell});
    } else {
        try writer.print("SHELL not set\n", .{});
    }

    // Environment variables with defaults
    const editor = init.process.getEnv("EDITOR") orelse "vi";
    try writer.print("Editor: {s}\n", .{editor});

    // Parse PATH into a list
    if (init.process.getEnv("PATH")) |path_var| {
        try writer.print("\nPATH directories:\n", .{});
        var it = std.mem.splitSequence(u8, path_var, ":");
        var count: usize = 0;
        while (it.next()) |dir| {
            count += 1;
            try writer.print("  {d}: {s}\n", .{count, dir});
        }
    }
}
```

### Setting Environment Variables for Child Processes

In Zig 0.16, you create a modified environment map and pass it when spawning a child process:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Get current environment
    var env_map = try init.process.getEnvMap(init.allocator);
    defer env_map.deinit(init.allocator);

    // Modify it
    try env_map.put(init.allocator, "MY_VAR", "hello_from_zig");
    try env_map.put(init.allocator, "GREETING", "Zig is awesome");

    try writer.print("Set MY_VAR and GREETING in child environment\n", .{});

    // Spawn a child process with the modified environment
    // (child will see the new variables)
    // Using std.process.Child:
    var child = std.process.Child.init(&.{"sh", "-c", "echo MY_VAR=$MY_VAR; echo GREETING=$GREETING"}, init.allocator);
    child.env_map = &env_map;

    const result = try child.spawnAndWait();
    try writer.print("Child exited with status: {d}\n", .{
        result.Exited orelse 1,
    });
}
```

---

## Chapter 4: File System Operations via std.Io

In Zig 0.16, all file system operations go through `std.Io`. The old `std.fs.cwd()` API has been replaced.

### Directory Listing

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Open the current working directory
    const cwd = std.Io.Dir.cwd();

    // List directory entries
    var iter = cwd.iterate();
    try writer.print("Current directory contents:\n", .{});

    var file_count: usize = 0;
    var dir_count: usize = 0;

    while (try iter.next()) |entry| {
        const kind_str = switch (entry.kind) {
            .file => "FILE",
            .directory => " DIR",
            .sym_link => "LINK",
            else => "????",
        };
        try writer.print("  [{s}] {s}\n", .{ kind_str, entry.name });

        if (entry.kind == .file) file_count += 1;
        if (entry.kind == .directory) dir_count += 1;
    }

    try writer.print("\n{d} files, {d} directories\n", .{ file_count, dir_count });
}
```

### Creating Files and Directories

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const cwd = std.Io.Dir.cwd();

    // Create a directory
    cwd.makeDir("test_dir") catch |err| {
        if (err != error.PathAlreadyExists) return err;
        try writer.print("Directory 'test_dir' already exists\n", .{});
    };

    // Create a nested directory (including parents)
    cwd.makePath("test_dir/nested/deep") catch |err| {
        if (err != error.PathAlreadyExists) return err;
    };
    try writer.print("Created test_dir/nested/deep\n", .{});

    // Create and write a file
    const file = try cwd.createFile("test_dir/nested/deep/hello.txt", .{});
    defer file.close();
    try file.writeAll("Hello from Zig 0.16!\n");
    try writer.print("Wrote to test_dir/nested/deep/hello.txt\n", .{});

    // Read the file back
    const content = try cwd.readFileAlloc(init.allocator, "test_dir/nested/deep/hello.txt", 1024);
    defer init.allocator.free(content);
    try writer.print("Read back: {s}", .{content});
}
```

### Renaming and Deleting

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const cwd = std.Io.Dir.cwd();

    // Create a file
    const f = try cwd.createFile("original.txt", .{});
    f.close();
    try writer.print("Created original.txt\n", .{});

    // Rename
    try cwd.rename("original.txt", "renamed.txt");
    try writer.print("Renamed to renamed.txt\n", .{});

    // Delete
    try cwd.deleteFile("renamed.txt");
    try writer.print("Deleted renamed.txt\n", .{});

    // Delete directory (must be empty)
    try cwd.deleteDir("test_dir/nested/deep");
    try writer.print("Deleted nested/deep\n", .{});
}
```

---

## Chapter 5: std.fs — Directories, Files, Paths

The `std.fs` module in Zig 0.16 primarily delegates to `std.Io`. The key types are:

### Absolute and Canonical Paths

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Current working directory (renamed API in 0.16)
    const cwd_path = try std.process.getCwd(init.allocator);
    defer init.allocator.free(cwd_path);
    try writer.print("Current directory: {s}\n", .{cwd_path});

    // Open an absolute directory
    const tmp_dir = try std.Io.Dir.openAt(.{}, "/tmp", .{});
    defer tmp_dir.close();

    try writer.print("Opened /tmp directory\n", .{});

    // List /tmp contents
    var iter = tmp_dir.iterate();
    var count: usize = 0;
    while (try iter.next()) |entry| {
        count += 1;
        if (count <= 10) {
            try writer.print("  {s}\n", .{entry.name});
        }
    }
    if (count > 10) {
        try writer.print("  ... and {d} more entries\n", .{count - 10});
    }
    try writer.print("Total: {d} entries in /tmp\n", .{count});
}
```

### File Metadata

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const cwd = std.Io.Dir.cwd();

    // Stat a file to get metadata
    const stat = try cwd.statFile("build.zig");

    try writer.print("build.zig metadata:\n", .{});
    try writer.print("  Size:    {d} bytes\n", .{stat.size});
    try writer.print("  Mode:    0o{o}\n", .{stat.mode});
    try writer.print("  Kind:    {s}\n", .{@tagName(stat.kind)});

    // Access and modification times
    const mtime = std.time.epoch.EpochSeconds{ .secs = @intCast(stat.mtime) };
    const epoch_day = mtime.getEpochDay();
    const day_info = epoch_day.getDayOfWeek();
    try writer.print("  Mtime weekday: {s}\n", .{@tagName(day_info)});
}
```

---

## Chapter 6: fs.path — Path Manipulation

Zig 0.16 includes `std.fs.path` for cross-platform path operations. Notably, `fs.path.relative` is now a **pure function** — it no longer accesses the filesystem.

### Common Path Operations

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();
    const path = std.fs.path;

    // Join paths (platform-aware separator)
    const joined = path.join("base", "dir", "file.txt");
    try writer.print("join: {s}\n", .{joined});

    // Using array
    const parts = [_][]const u8{ "a", "b", "c.txt" };
    const joined2 = path.join(&parts);
    try writer.print("join(&parts): {s}\n", .{joined2});

    // Basename (file name from path)
    const base = path.basename("/usr/local/bin/zig");
    try writer.print("basename: {s}\n", .{base});

    // Directory name
    const dir = path.dirname("/usr/local/bin/zig");
    try writer.print("dirname: {?s}\n", .{dir});

    // Extension
    const ext = path.extension("archive.tar.gz");
    try writer.print("extension: {s}\n", .{ext});

    // Resolve (normalize . and ..)
    const resolved = path.resolve("a/b/../c/./d");
    try writer.print("resolve: {s}\n", .{resolved});

    // relative is now PURE in 0.16 (no filesystem access)
    const rel = path.relative("/a/b/c", "/a/b/d/e.txt");
    try writer.print("relative: {s}\n", .{rel});

    const rel2 = path.relative("/a/b/c/d", "/a/b/c/e");
    try writer.print("relative: {s}\n", .{rel2});

    // isAbsolute
    try writer.print("isAbsolute('/foo'): {}\n", .{path.isAbsolute("/foo")});
    try writer.print("isAbsolute('foo'):  {}\n", .{path.isAbsolute("foo")});
}
```

### Windows Path Handling

`std.fs.path` automatically handles Windows path separators:

```zig
const path = std.fs.path;

// These work on all platforms:
_ = path.join("C:\\Users", "Documents"); // On Windows: C:\Users\Documents
_ = path.join("/home", "user");          // On POSIX: /home/user
```

---

## Chapter 7: std.time — Timestamps, Durations, Sleeping

### Instant and Duration

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Get current timestamp
    const now = std.time.Instant.now() catch return error.TimerFailed;
    try writer.print("Current instant: ns since epoch = {d}\n", .{now.ns});

    // Sleep for 1 second
    std.time.sleep(1 * std.time.ns_per_s);
    try writer.print("Slept for 1 second\n", .{});

    // Sleep for 500 milliseconds
    std.time.sleep(500 * std.time.ns_per_ms);
    try writer.print("Slept for 500ms\n", .{});

    // Measure elapsed time
    const start = std.time.Instant.now() catch return error.TimerFailed;

    // Do some work
    var sum: u64 = 0;
    for (0..1_000_000) |i| {
        sum +%= @intCast(i);
    }

    const end = std.time.Instant.now() catch return error.TimerFailed;
    const elapsed_ns = end.since(start);
    const elapsed_ms = @divTrunc(elapsed_ns, std.time.ns_per_ms);

    try writer.print("Computation took {d} ms (sum={d})\n", .{ elapsed_ms, sum });
}
```

### std.Io.Duration Format

Zig 0.16 introduces `std.Io.Duration`'s `format` method for human-readable duration output:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const start = std.time.Instant.now() catch return error.TimerFailed;
    std.time.sleep(2_567_890_123); // ~2.57 seconds
    const end = std.time.Instant.now() catch return error.TimerFailed;
    const elapsed = end.since(start);

    // Format as a Duration
    const duration: std.Io.Duration = @enumFromInt(elapsed);
    try writer.print("Elapsed: {}\n", .{duration});
    // Output will be a human-readable duration string
}
```

### Timestamp Conversion

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const timestamp = std.time.timestamp();

    // Convert to epoch days/hours/minutes/seconds
    const epoch_secs = std.time.epoch.EpochSeconds{ .secs = @intCast(timestamp) };
    const epoch_day = epoch_secs.getEpochDay();
    const day_year = epoch_day.getDayYear();
    const month_day = day_year.getMonthDay();

    const year = epoch_day.calculateYearDay().year;
    const month = month_day.month;
    const day = month_day.day_index + 1; // 0-indexed to 1-indexed

    const hours_minutes_seconds = epoch_secs.getDaySeconds();
    const hour = hours_minutes_seconds.getHoursIntoDay();
    const minute = hours_minutes_seconds.getMinutesIntoHour();
    const second = hours_minutes_seconds.getSecondsIntoMinute();

    try writer.print("Current UTC time: {d}-{d:0>2}-{d:0>2} {d:0>2}:{d:0>2}:{d:0>2}\n", .{
        year, @intFromEnum(month), day, hour, minute, second,
    });
}
```

---

## Chapter 8: std.process — Getting Process Info

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Process ID
    const pid = std.posix.system.getpid();
    try writer.print("PID:  {d}\n", .{pid});

    // Parent Process ID
    const ppid = std.posix.system.getppid();
    try writer.print("PPID: {d}\n", .{ppid});

    // Process arguments
    const args = try init.process.argsAlloc(init.allocator);
    defer init.process.argsFree(init.allocator, args);

    try writer.print("\nArguments ({d}):\n", .{args.len});
    for (args, 0..) |arg, i| {
        try writer.print("  [{d}] {s}\n", .{i, arg});
    }

    // Executable name
    if (args.len > 0) {
        try writer.print("\nExecutable: {s}\n", .{args[0]});
    }

    // Current working directory (renamed API in 0.16)
    const cwd = try std.process.getCwd(init.allocator);
    defer init.allocator.free(cwd);
    try writer.print("Working dir: {s}\n", .{cwd});

    // User ID and Group ID
    const uid = std.posix.system.getuid();
    const gid = std.posix.system.getgid();
    try writer.print("UID: {d}, GID: {d}\n", .{uid, gid});
}
```

### Spawning Child Processes

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    // Simple child process
    var child = std.process.Child.init(&.{
        "echo", "Hello from child process!",
    }, init.allocator);

    child.stdout_behavior = .Pipe;
    child.stderr_behavior = .Inherit;

    try child.spawn();

    // Read child's stdout
    const stdout_result = try child.stdout.?.reader().readAllAlloc(init.allocator, 1024);
    defer init.allocator.free(stdout_result);

    const term = try child.wait();
    try writer.print("Child output: {s}", .{stdout_result});
    try writer.print("Child exited: {s}\n", .{@tagName(term)});
}
```

---

## Chapter 9: std.Target — Compile-Time Target Detection

`std.Target` provides rich compile-time information about the target platform.

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const target = comptime std.Target.current;

    // OS information
    try writer.print("=== Operating System ===\n", .{});
    try writer.print("Tag:         {s}\n", .{@tagName(target.os.tag)});
    try writer.print("Version:     {s}\n", .{target.os.version_range.semver.min.toString()});
    try writer.print("ABI:         {s}\n", .{@tagName(target.abi)});

    // CPU information
    try writer.print("\n=== CPU ===\n", .{});
    try writer.print("Arch:        {s}\n", .{@tagName(target.cpu.arch)});
    try writer.print("Endianness:  {s}\n", .{@tagName(target.cpu.arch.endian())});
    try writer.print("Has AES NI:  {}\n", .{target.cpu.features.isEnabled(@ptrFromInt(0))});

    // Useful comptime booleans
    const is_64bit = target.ptrBitWidth() == 64;
    const is_little = target.cpu.arch.endian() == .little;
    const is_linux = target.os.tag == .linux;
    const is_x86 = target.cpu.arch == .x86_64 or target.cpu.arch == .x86;

    try writer.print("\n=== Derived ===\n", .{});
    try writer.print("64-bit:      {}\n", .{is_64bit});
    try writer.print("Little-Endian: {}\n", .{is_little});
    try writer.print("Linux:       {}\n", .{is_linux});
    try writer.print("x86/x86_64:  {}\n", .{is_x86});
}
```

### Using Target for Conditional Compilation

```zig
const std = @import("std");

// Comptime function that generates platform-specific code
fn platformByteSwap(value: u32) u32 {
    return switch (comptime std.Target.current.cpu.arch.endian()) {
        .little => @byteSwap(value),
        .big => value,
    };
}

// Comptime selection of atomic operations
fn atomicLoad(comptime T: type, ptr: *const T) T {
    if (comptime std.Target.current.cpu.arch == .x86_64) {
        return @atomicLoad(T, ptr, .seq_cst);
    } else {
        return @atomicLoad(T, ptr, .seq_cst);
    }
}

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    const val: u32 = 0x12345678;
    const swapped = platformByteSwap(val);
    try writer.print("Original:  0x{x}\n", .{val});
    try writer.print("Swapped:   0x{x}\n", .{swapped});
}
```

---

## Chapter 10: Platform-Specific Code with Comptime

Zig's `comptime` is the primary mechanism for writing platform-specific code without preprocessor macros.

### Operating System Abstraction Layer

```zig
const std = @import("std");

const OsLayer = struct {
    // Each field is a function pointer or comptime-known value
    const current = switch (comptime std.Target.current.os.tag) {
        .linux => LinuxImpl,
        .macos => MacImpl,
        .windows => WindowsImpl,
        else => GenericImpl,
    };

    fn getPageSize() usize {
        return current.getPageSize();
    }

    fn getCpuCount() usize {
        return current.getCpuCount();
    }

    fn getHomeDir() []const u8 {
        return current.getHomeDir();
    }

    const LinuxImpl = struct {
        fn getPageSize() usize { return 4096; }
        fn getCpuCount() usize { return @import("std").Thread.CpuCount.detect(); }
        fn getHomeDir() []const u8 { return "/home/user"; }
    };

    const MacImpl = struct {
        fn getPageSize() usize { return 16384; }
        fn getCpuCount() usize { return @import("std").Thread.CpuCount.detect(); }
        fn getHomeDir() []const u8 { return "/Users/user"; }
    };

    const WindowsImpl = struct {
        fn getPageSize() usize { return 4096; }
        fn getCpuCount() usize { return @import("std").Thread.CpuCount.detect(); }
        fn getHomeDir() []const u8 { return "C:\\Users\\user"; }
    };

    const GenericImpl = struct {
        fn getPageSize() usize { return 4096; }
        fn getCpuCount() usize { return 1; }
        fn getHomeDir() []const u8 { return "/"; }
    };
};

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();

    try writer.print("Page size:   {d}\n", .{OsLayer.getPageSize()});
    try writer.print("CPU count:   {d}\n", .{OsLayer.getCpuCount()});
    try writer.print("Home dir:    {s}\n", .{OsLayer.getHomeDir()});
}
```

### Using build options for platform features

In `build.zig`, you can expose platform-specific options:

```zig
// build.zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const use_epoll = b.option(bool, "use_epoll", "Use epoll on Linux") orelse
        (target.result.os.tag == .linux);

    const exe = b.addExecutable(.{
        .name = "sys-mon",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    exe.root_module.addOption(bool, "use_epoll", use_epoll);
    b.installArtifact(exe);
}
```

```zig
// In your Zig code:
const use_epoll = @import("builtin").option_bool_use_epoll;
```

---

## Chapter 11: Project — Cross-Platform System Monitor

This project creates a system monitor that displays CPU count, memory info, process info, and environment details, adapting to the host OS.

### build.zig

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "sys-mon",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });
    b.installArtifact(exe);

    const run_cmd = b.addRunArtifact(exe);
    run_cmd.step.dependOn(b.getInstallStep());
    b.step("run", "Run the system monitor").dependOn(&run_cmd.step);
}
```

### src/main.zig

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const stdout = init.io.getStdOut();
    const writer = stdout.writer();
    const target = comptime std.Target.current;

    try writer.print("╔══════════════════════════════════════════╗\n", .{});
    try writer.print("║       Zig 0.16 System Monitor           ║\n", .{});
    try writer.print("╚══════════════════════════════════════════╝\n", .{});

    // System info
    try writer.print("\n── System Information ──\n", .{});
    try writer.print("  OS:        {s}\n", .{@tagName(target.os.tag)});
    try writer.print("  Arch:      {s}\n", .{@tagName(target.cpu.arch)});
    try writer.print("  Endian:    {s}\n", .{@tagName(target.cpu.arch.endian())});
    try writer.print("  Pointer:   {d}-bit\n", .{target.ptrBitWidth()});

    // Process info
    try writer.print("\n── Process Information ──\n", .{});
    const pid = std.posix.system.getpid();
    const ppid = std.posix.system.getppid();
    try writer.print("  PID:       {d}\n", .{pid});
    try writer.print("  PPID:      {d}\n", .{ppid});

    const uid = std.posix.system.getuid();
    const gid = std.posix.system.getgid();
    try writer.print("  UID:       {d}\n", .{uid});
    try writer.print("  GID:       {d}\n", .{gid});

    // Working directory
    const cwd = try std.process.getCwd(init.allocator);
    defer init.allocator.free(cwd);
    try writer.print("  CWD:       {s}\n", .{cwd});

    // Command line
    const args = try init.process.argsAlloc(init.allocator);
    defer init.process.argsFree(init.allocator, args);
    try writer.print("  Exe:       {s}\n", .{if (args.len > 0) args[0] else "(unknown)"});

    // Environment summary
    try writer.print("\n── Environment Summary ──\n", .{});
    const env_map = try init.process.getEnvMap(init.allocator);
    defer env_map.deinit(init.allocator);
    try writer.print("  Variables: {d}\n", .{env_map.count()});

    // Show selected variables
    const important_vars = [_][]const u8{ "HOME", "USER", "SHELL", "PATH", "LANG", "TERM" };
    for (important_vars) |var_name| {
        if (env_map.get(var_name)) |val| {
            const display = if (val.len > 40) val[0..40] ++ "..." else val;
            try writer.print("  {s: >8} = {s}\n", .{ var_name, display });
        }
    }

    // File system info
    try writer.print("\n── File System ──\n", .{});
    const cwd_dir = std.Io.Dir.cwd();

    var file_count: usize = 0;
    var dir_count: usize = 0;
    var total_size: u64 = 0;

    var iter = cwd_dir.iterate();
    while (try iter.next()) |entry| {
        switch (entry.kind) {
            .file => file_count += 1,
            .directory => dir_count += 1,
            else => {},
        }
        // Get size for files
        if (entry.kind == .file) {
            const stat = cwd_dir.statFile(entry.name) catch continue;
            total_size += stat.size;
        }
    }
    try writer.print("  Files:     {d}\n", .{file_count});
    try writer.print("  Dirs:      {d}\n", .{dir_count});
    try writer.print("  Total:     {d} bytes ({d:.1} KB)\n", .{
        total_size,
        @as(f64, @floatFromInt(total_size)) / 1024.0,
    });

    // Path operations demo
    try writer.print("\n── Path Operations ──\n", .{});
    const path = std.fs.path;
    try writer.print("  basename({s}): {s}\n", .{ cwd, path.basename(cwd) });
    try writer.print("  dirname({s}):  {?s}\n", .{ cwd, path.dirname(cwd) });
    try writer.print("  isAbsolute({s}): {}\n", .{ cwd, path.isAbsolute(cwd) });

    // Time info
    try writer.print("\n── Time ──\n", .{});
    const timestamp = std.time.timestamp();
    const epoch_secs = std.time.epoch.EpochSeconds{ .secs = @intCast(timestamp) };
    const day_year = epoch_secs.getEpochDay().getDayYear();
    const month_day = day_year.getMonthDay();
    const year = epoch_secs.getEpochDay().calculateYearDay().year;
    const hms = epoch_secs.getDaySeconds();

    try writer.print("  UTC:       {d}-{d:0>2}-{d:0>2} {d:0>2}:{d:0>2}:{d:0>2}\n", .{
        year, @intFromEnum(month_day.month), month_day.day_index + 1,
        hms.getHoursIntoDay(), hms.getMinutesIntoHour(), hms.getSecondsIntoMinute(),
    });

    // Memory measurement
    try writer.print("\n── Performance ──\n", .{});
    const start = std.time.Instant.now() catch return error.TimerFailed;
    var total: u64 = 0;
    for (0..10_000_000) |i| {
        total +%= @intCast(i);
    }
    const end = std.time.Instant.now() catch return error.TimerFailed;
    const elapsed_us = @divTrunc(end.since(start), std.time.ns_per_us);
    try writer.print("  Loop 10M:  {d} μs\n", .{elapsed_us});
    try writer.print("  Sum:       {d}\n", .{total});

    try writer.print("\n══════════════════════════════════════════\n", .{});
}
```

### Sample Output

```
╔══════════════════════════════════════════╗
║       Zig 0.16 System Monitor           ║
╚══════════════════════════════════════════╝

── System Information ──
  OS:        linux
  Arch:      x86_64
  Endian:    little
  Pointer:   64-bit

── Process Information ──
  PID:       12345
  PPID:      1000
  UID:       1000
  GID:       1000
  CWD:       /home/user/my-project
  Exe:       ./zig-out/bin/sys-mon

── Environment Summary ──
  Variables: 42
     HOME = /home/user
     USER = user
    SHELL = /bin/bash
     PATH = /usr/local/bin:/usr/bin:/bin
     LANG = en_US.UTF-8
     TERM = xterm-256color

── File System ──
  Files:     12
  Dirs:      3
  Total:     45678 bytes (44.6 KB)

── Time ──
  UTC:       2025-01-15 14:30:45

── Performance ──
  Loop 10M:  12450 μs
  Sum:       49999995000000

══════════════════════════════════════════
```

---

## Summary

| Module | Purpose | 0.16 Notes |
|---|---|---|
| `std.process` | Args, env vars, cwd, child processes | Env vars are non-global |
| `std.Io` | Files, directories, I/O | Replaces mid-level `std.os`/`std.posix` |
| `std.fs.path` | Path manipulation | `relative` is now pure |
| `std.time` | Timestamps, sleeping, instants | New `std.Io.Duration` format |
| `std.posix` | Types, constants, syscalls | Mid-level functions removed |
| `std.Target` | Compile-time target detection | CPU, OS, ABI info |
| `std.builtin` | Built-in compile-time info | `builtin.os.tag`, `builtin.cpu.arch` |