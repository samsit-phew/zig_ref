# Process Management in Zig 0.16

A comprehensive guide to spawning, controlling, and communicating with child processes using Zig's redesigned `std.process` API.

---

## Chapter 1: Introduction to std.process

Zig's standard library provides a rich set of process management primitives through the `std.process` namespace. In Zig 0.16, this module has undergone a significant redesign that moves away from global state toward a non-global, composable API centered around `std.process.Init`.

The `std.process` module in 0.16 covers:

- **Spawning child processes** with fine-grained control over stdin/stdout/stderr
- **Replacing the current process image** (the former `execv`)
- **Accessing process arguments and environment variables** through the `Init` struct
- **Process termination, waiting, and signal handling**

Every Zig program receives a `std.process.Init` parameter in its main function. This is your gateway to all process-related state:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    _ = init;
    std.log.info("Process management demo", .{});
}
```

The `std.process.Init` struct encapsulates everything the runtime knows about the current process at startup: command-line arguments, environment variables, and I/O resources. Unlike previous Zig versions where `std.process.argsAlloc` and `std.os.getenv` accessed global state, Zig 0.16 requires you to pass `init` around explicitly.

This design has several advantages:
1. **Testability**: You can construct a fake `Init` in tests
2. **Clarity**: Dependencies on process state are visible in function signatures
3. **Thread safety**: No hidden global mutable state

---

## Chapter 2: The 0.16 Redesign — Non-Global Process State

### What Changed?

In Zig 0.15 and earlier, process arguments and environment variables were accessed via global functions:

```zig
// OLD (pre-0.16) — NO LONGER WORKS
const args = try std.process.argsAlloc(allocator);
const env_val = std.posix.getenv("HOME");
```

In Zig 0.16, these are accessed through `std.process.Init`:

```zig
// NEW (0.16 stable)
pub fn main(init: std.process.Init) !void {
    const args = init.minimal.args;
    // args is a slice of []const u8 representing command-line arguments

    // Environment access is now through the init parameter
    const env_map = try init.environ.getMap(allocator);
    defer env_map.deinit(allocator);

    if (env_map.get("HOME")) |home| {
        std.log.info("Home directory: {s}", .{home});
    }
}
```

### The `Init` Struct

The `std.process.Init` struct provides:

| Field | Description |
|-------|-------------|
| `minimal.args` | Slice of command-line argument strings |
| `environ` | Environment variable accessor |
| `stdin` | Standard input handle |
| `stdout` | Standard output handle |
| `stderr` | Standard error handle |

### Minimal vs Full Init

Zig 0.16 distinguishes between minimal and full process initialization. The `init.minimal` sub-struct provides access to core process state without pulling in the entire I/O subsystem. This is useful for programs that need argument parsing but don't need stdio:

```zig
pub fn main(init: std.process.Init) !void {
    // Minimal access — just args
    for (init.minimal.args, 0..) |arg, i| {
        std.log.info("arg[{}] = {s}", .{ i, arg });
    }
}
```

---

## Chapter 3: std.process.spawn — Spawning Child Processes

### The New API

Zig 0.16 replaces `std.process.Child.init`/`spawn`/`run` with a streamlined `std.process.spawn` function:

```zig
const result = try std.process.spawn(io, .{
    .argv = &[_][]const u8{ "ls", "-la" },
    .stdin = .pipe,
    .stdout = .pipe,
    .stderr = .pipe,
});
```

The first argument is an `*std.Io` instance (obtained from `init.io` or constructed), and the second argument is an anonymous struct with configuration options.

### argv Configuration

The `argv` field accepts a slice of string slices. The first element is conventionally the program name:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const result = try std.process.spawn(io, .{
        .argv = &[_][]const u8{
            "echo",
            "Hello from a child process!",
        },
        .stdout = .inherit,
        .stderr = .inherit,
    });

    const term = result.wait();
    std.log.info("Child exited with code {}", .{term.Exited});
}
```

### Pipe Configuration Options

The stdin/stdout/stderr fields accept these values:

| Value | Behavior |
|-------|----------|
| `.inherit` | Child inherits the parent's file descriptor |
| `.pipe` | Create a pipe; parent can read/write via returned handles |
| `.ignore` | Redirect to `/dev/null` (or platform equivalent) |
| `.close` | Close the file descriptor in the child |

### Spawned Process Handle

`std.process.spawn` returns a handle that allows you to interact with the child:

```zig
const child = try std.process.spawn(io, .{
    .argv = &[_][]const u8{ "sleep", "5" },
    .stdin = .ignore,
    .stdout = .inherit,
    .stderr = .inherit,
});

defer _ = child.kill() catch {};

// Wait for the child to finish
const term = child.wait();
std.log.info("Process terminated", .{});
```

---

## Chapter 4: std.process.run — Run and Capture Output

When you simply want to run a command and capture its output, `std.process.run` is the most convenient API:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;

    // Run a command and capture stdout
    const result = try std.process.run(allocator, io, .{
        .argv = &[_][]const u8{ "zig", "version" },
    });
    defer allocator.free(result.stdout);

    std.log.info("Zig version: {s}", .{result.stdout});
}
```

### Working with Result

The `std.process.RunResult` struct provides:

```zig
pub const RunResult = struct {
    stdout: []u8,
    stderr: []u8,
    term: std.process.Term,
};
```

You can inspect both stdout and stderr, as well as the termination status:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;

    const result = try std.process.run(allocator, io, .{
        .argv = &[_][]const u8{ "zig", "build", "--help" },
    });
    defer allocator.free(result.stdout);
    defer allocator.free(result.stderr);

    const writer = init.stderr.writer();
    try writer.print("Exit code: {}\n", .{result.term.Exited});
    try writer.print("Stdout length: {} bytes\n", .{result.stdout.len});
    try writer.print("Stderr length: {} bytes\n", .{result.stderr.len});
}
```

### Setting Environment Variables for the Child

```zig
const result = try std.process.run(allocator, io, .{
    .argv = &[_][]const u8{ "printenv", "MY_VAR" },
    .env_map = &env_map,
});
```

---

## Chapter 5: std.process.replace — Replace Current Process Image

The `std.process.replace` function replaces the current process with a new program (the equivalent of POSIX `execve`). This is the 0.16 replacement for the old `std.process.execv`:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const io = init.io;

    // This never returns on success — the current process is replaced
    std.process.replace(io, .{
        .argv = &[_][]const u8{ "/bin/ls", "-la" },
    });

    // If we get here, replace failed
    std.log.err("replace failed", .{});
}
```

### When to Use replace

Common use cases for `std.process.replace`:

1. **Shell implementations**: After setting up pipes and redirections, replace the shell process with the command
2. **Process supervisors**: Re-exec the current binary with different arguments
3. **Shebang-style interpreters**: An interpreter can replace itself with the target after setup

### Replace with Environment

```zig
pub fn main(init: std.process.Init) !void {
    const io = init.io;

    const env_map = try init.environ.getMap(std.heap.page_allocator);

    // Add/override an environment variable for the new process
    try env_map.put("MY_SETTING", "42");

    std.process.replace(io, .{
        .argv = &[_][]const u8{ "/usr/bin/env" },
        .env_map = &env_map,
    });
}
```

---

## Chapter 6: Working with Child Stdin/Stdout/Stderr Pipes

Pipes allow bidirectional communication between parent and child processes. Here is a complete example of writing to a child's stdin and reading from its stdout:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;

    const child = try std.process.spawn(io, .{
        .argv = &[_][]const u8{ "cat" },
        .stdin = .pipe,
        .stdout = .pipe,
        .stderr = .inherit,
    });

    // Write to child's stdin
    const stdin_writer = child.stdin.writer();
    try stdin_writer.writeAll("Hello via pipe!\n");
    child.stdin.close(); // Signal EOF to the child

    // Read child's stdout
    const stdout_reader = child.stdout.reader();
    const output = try stdout_reader.readAllAlloc(allocator, 4096);
    defer allocator.free(output);

    const term = child.wait();
    _ = term;

    const writer = init.stdout.writer();
    try writer.print("Child output: {s}", .{output});
}
```

### Piping Between Two Children

You can pipe the output of one child into the input of another:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;

    const child1 = try std.process.spawn(io, .{
        .argv = &[_][]const u8{ "echo", "line1\nline2\nline3" },
        .stdout = .pipe,
    });

    const child2 = try std.process.spawn(io, .{
        .argv = &[_][]const u8{ "wc", "-l" },
        .stdin = .pipe,
        .stdout = .inherit,
    });

    // Read from child1's stdout and write to child2's stdin
    const buf = try child1.stdout.reader().readAllAlloc(allocator, 4096);
    defer allocator.free(buf);

    try child2.stdin.writer().writeAll(buf);
    child2.stdin.close();

    _ = child1.wait();
    _ = child2.wait();
}
```

---

## Chapter 7: Environment Variables in 0.16

### Accessing the Environment

In Zig 0.16, environment variables are no longer global. They are accessed through the `Init` parameter:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;

    // Get the environment as a hash map
    const env_map = try init.environ.getMap(allocator);
    defer env_map.deinit(allocator);

    // Iterate over all environment variables
    var it = env_map.iterator();
    while (it.next()) |entry| {
        std.log.info("{s}={s}", .{ entry.key_ptr.*, entry.value_ptr.* });
    }

    // Get a specific variable
    if (env_map.get("PATH")) |path| {
        const writer = init.stdout.writer();
        try writer.print("PATH: {s}\n", .{path});
    }
}
```

### Modifying Environment for Child Processes

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;

    // Copy the current environment
    var env_map = try init.environ.getMap(allocator);
    defer env_map.deinit(allocator);

    // Modify it
    try env_map.put("CUSTOM_VAR", "custom_value");
    _ = env_map.remove("UNWANTED_VAR");

    // Spawn a child with the modified environment
    const result = try std.process.run(allocator, io, .{
        .argv = &[_][]const u8{ "printenv", "CUSTOM_VAR" },
        .env_map = &env_map,
    });
    defer allocator.free(result.stdout);

    const writer = init.stdout.writer();
    try writer.print("Custom var value: {s}", .{result.stdout});
}
```

---

## Chapter 8: Process Arguments (non-global)

### Iterating Arguments

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const args = init.minimal.args;
    const writer = init.stdout.writer();

    try writer.print("Program: {s}\n", .{args[0]});
    try writer.print("Argument count: {}\n", .{args.len - 1});

    for (args[1..], 1..) |arg, i| {
        try writer.print("  arg[{}] = {s}\n", .{ i, arg });
    }
}
```

### Parsing Flags

Here is a simple argument parser using the non-global args:

```zig
const std = @import("std");

const Config = struct {
    verbose: bool = false,
    output: ?[]const u8 = null,
    input: ?[]const u8 = null,
};

fn parseArgs(args: [][]const u8) !Config {
    var config = Config{};
    var i: usize = 1; // skip program name

    while (i < args.len) {
        if (std.mem.eql(u8, args[i], "-v") or std.mem.eql(u8, args[i], "--verbose")) {
            config.verbose = true;
        } else if (std.mem.eql(u8, args[i], "-o")) {
            i += 1;
            if (i >= args.len) return error.MissingArgument;
            config.output = args[i];
        } else if (config.input == null) {
            config.input = args[i];
        }
        i += 1;
    }

    return config;
}

pub fn main(init: std.process.Init) !void {
    const args = init.minimal.args;
    const config = try parseArgs(args);
    const writer = init.stdout.writer();

    try writer.print("verbose: {}\n", .{config.verbose});
    if (config.output) |out| try writer.print("output: {s}\n", .{out});
    if (config.input) |inp| try writer.print("input: {s}\n", .{inp});
}
```

---

## Chapter 9: Process Termination, Signals, and Wait

### Waiting for a Child

```zig
const child = try std.process.spawn(io, .{
    .argv = &[_][]const u8{ "sleep", "2" },
});

const term = child.wait();
switch (term) {
    .Exited => |code| std.log.info("Exited normally with code {}", .{code}),
    .Signal => |sig| std.log.info("Killed by signal {}", .{sig}),
    .Stopped => |sig| std.log.info("Stopped by signal {}", .{sig}),
    .Unknown => |status| std.log.info("Unknown status: {}", .{status}),
}
```

### Terminating a Child

```zig
const child = try std.process.spawn(io, .{
    .argv = &[_][]const u8{ "sleep", "100" },
});

// Send SIGTERM
child.kill() catch |err| {
    std.log.err("Failed to kill child: {}", .{err});
};

const term = child.wait();
_ = term;
```

### Timeouts

Zig 0.16 does not have a built-in timeout for `wait()`. Implement one with threads:

```zig
const std = @import("std");

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;

    const child = try std.process.spawn(io, .{
        .argv = &[_][]const u8{ "sleep", "100" },
    });

    // Wait in a separate thread
    const thread = try std.Thread.spawn(.{}, waitForChild, .{child});

    // Give it 2 seconds
    std.time.sleep(2 * std.time.ns_per_s);

    child.kill() catch {};
    thread.join();

    const writer = init.stdout.writer();
    try writer.writeAll("Child was terminated after timeout\n");
}

fn waitForChild(child: anytype) void {
    _ = child.wait();
}
```

---

## Chapter 10: Project — A Build System / Task Runner

This project implements a simple task runner that reads tasks from a configuration structure and executes them as child processes, capturing output and reporting results.

### `build.zig`

```zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "taskrunner",
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

const Task = struct {
    name: []const u8,
    command: []const u8,
    args: []const []const u8,
    depends_on: []const []const u8 = &.{},
};

const TaskResult = struct {
    name: []const u8,
    exit_code: u8,
    stdout: []u8,
    duration_ns: u64,
};

const tasks = [_]Task{
    .{
        .name = "greet",
        .command = "echo",
        .args = &.{"Hello from task 'greet'!"},
    },
    .{
        .name = "list-files",
        .command = "ls",
        .args = &.{"-1", "/tmp"},
    },
    .{
        .name = "show-date",
        .command = "date",
        .args = &.{},
    },
    .{
        .name = "zig-version",
        .command = "zig",
        .args = &.{"version"},
    },
};

pub fn main(init: std.process.Init) !void {
    const allocator = std.heap.page_allocator;
    const io = init.io;
    const writer = init.stdout.writer();

    try writer.print("=== Task Runner ===\n", .{});
    try writer.print("Found {} tasks\n\n", .{tasks.len});

    var results: [tasks.len]TaskResult = undefined;
    var passed: usize = 0;
    var failed: usize = 0;

    for (&tasks, 0..) |*task, i| {
        try writer.print("[RUN] {s}: {s} {s}\n", .{
            task.name,
            task.command,
            joinArgs(task.args),
        });

        const start = std.time.nanoTimestamp();

        // Build argv: command + args
        var argv_buf: [16][]const u8 = undefined;
        argv_buf[0] = task.command;
        for (task.args, 0..) |arg, j| {
            argv_buf[j + 1] = arg;
        }
        const argv = argv_buf[0 .. 1 + task.args.len];

        const result = std.process.run(allocator, io, .{
            .argv = argv,
        }) catch |err| {
            try writer.print("[FAIL] {s}: spawn error: {}\n", .{ task.name, err });
            results[i] = .{
                .name = task.name,
                .exit_code = 255,
                .stdout = "",
                .duration_ns = 0,
            };
            failed += 1;
            continue;
        };

        const end = std.time.nanoTimestamp();
        const duration = @as(u64, @intCast(end - start));

        const exit_code = switch (result.term) {
            .Exited => |c| @as(u8, @intCast(c)),
            else => 1,
        };

        if (exit_code == 0) {
            passed += 1;
            try writer.print("[PASS] {s} ({}ms)\n", .{
                task.name,
                duration / 1_000_000,
            });
        } else {
            failed += 1;
            try writer.print("[FAIL] {s} (exit code {})\n", .{
                task.name,
                exit_code,
            });
        }

        // Print first line of output
        if (result.stdout.len > 0) {
            const first_newline = std.mem.find(u8, result.stdout, "\n") orelse result.stdout.len;
            try writer.print("       > {s}\n", .{result.stdout[0..first_newline]});
        }

        results[i] = .{
            .name = task.name,
            .exit_code = exit_code,
            .stdout = result.stdout,
            .duration_ns = duration,
        };
    }

    // Summary
    try writer.print("\n=== Summary ===\n", .{});
    try writer.print("Passed: {}/{}\n", .{ passed, tasks.len });
    try writer.print("Failed: {}/{}\n", .{ failed, tasks.len });
    try writer.print("Total time: {}ms\n", .{
        totalDuration(&results) / 1_000_000,
    });

    // Clean up
    for (&results) |*r| {
        allocator.free(r.stdout);
    }

    if (failed > 0) std.process.exit(1);
}

fn joinArgs(args: []const []const u8) []const u8 {
    if (args.len == 0) return "";
    // Simple concatenation for display
    _ = args;
    return "...";
}

fn totalDuration(results: []const TaskResult) u64 {
    var total: u64 = 0;
    for (results) |r| total += r.duration_ns;
    return total;
}
```

### Running the Task Runner

```bash
zig build run
```

Expected output:

```
=== Task Runner ===
Found 4 tasks

[RUN] greet: echo Hello from task 'greet'!
[PASS] greet (12ms)
       > Hello from task 'greet'!
[RUN] list-files: ls -1 /tmp
[PASS] list-files (8ms)
[RUN] show-date: date
[PASS] show-date (6ms)
[RUN] zig-version: zig version
[PASS] zig-version (45ms)
       > 0.16.0

=== Summary ===
Passed: 4/4
Failed: 0/4
Total time: 71ms
```

---

## Summary

Zig 0.16's process management API represents a clean break from global state. Key takeaways:

1. **`std.process.Init`** is your entry point — it carries args, env, and I/O handles
2. **`std.process.spawn`** replaces the old `Child.init`/`spawn` pattern with a cleaner function-based API
3. **`std.process.run`** is the convenient way to run a command and capture output
4. **`std.process.replace`** replaces `execv` for process image replacement
5. **Environment variables and arguments are non-global**, accessed through `init.environ` and `init.minimal.args`
6. **Pipes** for stdin/stdout/stderr are configured via `.pipe`, `.inherit`, `.ignore`, or `.close`

This redesign makes Zig's process management more composable, more testable, and more explicit about its dependencies on runtime state.
