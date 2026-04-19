<div align="center" style="text-align: justify;">

```  _   ___     ____   __
 | \ | \ \   / /\ \ / /
 |  \| |\ \_/ /  \ V / 
 | . ` | \   /    > <  
 | |\  |  | |    / . \ 
 |_| \_|  |_|   /_/ \_\

```
Nyx is a low-level, statically typed systems programming language

[![CI](https://github.com/noirAnayCO/nyx/actions/workflows/ci.yml/badge.svg)](https://github.com/noirAnayCO/nyx/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Status: Early Development](https://img.shields.io/badge/status-early%20development-orange)](#)

</div>
_____

# Nyx Language — Product Requirements Document

**Version:** 0.3 (Expanded)
**Status:** Draft

-----

## 1. Overview

Nyx is a low-level, statically typed systems programming language designed for developers who need direct machine control without sacrificing clarity or safety. It draws from C’s performance model and Rust’s safety philosophy, but charts its own path: smaller, more predictable, and without the learning overhead of a full borrow checker.

Nyx is built to be self-hosting. The end goal is a compiler written in Nyx that compiles Nyx — starting from a bootstrapped x86-64 backend and growing into a complete toolchain.

-----

## 2. Goals

- Deliver C-level performance and hardware control.
- Reduce accidental memory bugs without mandating a heavy ownership model.
- Stay small, explicit, and learnable — no language magic.
- Target real use cases from day one: kernel development, CLI/TUI tools, ML frameworks, and a self-hosting compiler.
- First-class C interoperability with zero friction.
- Compile directly to x86-64 assembly with no intermediate runtime dependency.

-----

## 3. Non-Goals

- Not a scripting language. No REPL-first design.
- Not a Rust clone. Nyx does not implement a full borrow checker.
- Not garbage collected. Memory is always explicitly managed.
- Not ecosystem-first. The standard library stays minimal at launch.
- No virtual machine or bytecode layer in the core path.

-----

## 4. Target Use Cases (Priority Order)

|Priority|Domain               |Notes                                                       |
|--------|---------------------|------------------------------------------------------------|
|1       |Kernel / OS dev      |Requires bare-metal mode, no runtime, direct hardware access|
|2       |Self-hosting compiler|Nyx compiler written in Nyx                                 |
|3       |CLI + TUI tooling    |Lightweight, fast startup, expressive                       |
|4       |ML frameworks        |Manual SIMD, raw memory layout control                      |

-----

## 5. Core Design Principles

- **Explicit over implicit.** The programmer always knows what is happening.
- **Safe by default.** Unsafe operations require deliberate opt-in.
- **Unsafe is visible.** All raw pointer work lives inside marked blocks or annotations.
- **Minimal runtime.** The language can run with zero runtime support.
- **Fast compilation.** Single-pass where possible. No slow linker magic.
- **Predictable behavior.** No hidden allocations, no surprise copies, no implicit conversions.
- **Dual syntax, one language.** Two syntax modes that interoperate freely within a single file.
- **You can always see what the machine is doing.** Reading Nyx code tells you exactly what happens at runtime — no hidden costs.

-----

## 6. Transparency Principle

### 6.1 The C vs C++ Problem

Nyx has a hard design rule inspired by a clear real-world contrast: C code is readable because every operation maps directly to something the machine does. C++ code is often unreadable because operators like `<<` are overloaded, constructors run silently, copies happen implicitly, and abstraction layers stack on top of each other until you can no longer tell what the CPU is actually executing.

Nyx explicitly rejects that path.

**The rule is simple:** if you can read a line of Nyx and not know what the hardware is doing, that is a language design bug.

### 6.2 What This Means Concretely

**No operator overloading.**
Operators in Nyx mean exactly what they say. `+` is addition. `*` is multiply or dereference. `<<` is a bitshift — always. You will never see `<<` used to push to a stream. If a type needs a custom operation, it gets a named method.

```nyx
// ❌ Never in Nyx — this is what ruins C++ readability
cout << "hello" << endl;

// ✅ Nyx — you know exactly what this is
write(stdout, "hello\n", 6);
```

**No implicit conversions.**
Types do not silently coerce. If you want an `i64` from an `i32`, you cast it explicitly. The compiler never widens, narrows, or promotes a value without source-level evidence.

```nyx
x: i32 = 42;
y: i64 = x as i64;   // explicit — you see the conversion
```

**No hidden allocations.**
If memory is being allocated, `alloc` appears in the source. No constructor silently heap-allocates. No string concatenation secretly calls `malloc`. Every byte of dynamic memory has a visible source origin.

**No hidden function calls.**
Nyx has no constructors, destructors, or implicit copy/move functions that run without appearing in source. When a function is called, it is written in the source. Period.

```nyx
// ❌ C++ — destructor called silently at scope exit, allocation hidden
{
    std::string s = "hello";
}   // ~string() called here, heap freed — invisible

// ✅ Nyx — everything visible
{
    own s: *char = alloc(6);
    copy("hello", s, 6);
}   // compiler inserts free(s) here — you know it happens because you wrote `own`
```

**No magic methods.**
There are no special method names the compiler looks for (`__init__`, `operator=`, `Drop`, etc.). Behaviour is not attached to naming conventions. If the language does something, it uses an explicit keyword — not a naming contract.

**Methods are just functions with a receiver.**
When you call `vec.len()`, `len` is a plain function that takes a pointer to `vec` as its first argument. There is no vtable lookup, no dynamic dispatch, no magic — unless you explicitly use `dyn` (post-MVP). The `.` operator is syntactic sugar for a normal function call, nothing more.

```nyx
// These are identical:
vec.push(5);
Vec_push(&vec, 5);
```

### 6.3 What Nyx Does Allow

Nyx is not hostile to abstraction — it is hostile to *invisible* abstraction. You can build high-level constructs in Nyx. You can write clean, readable, expressive code. The constraint is that every abstraction must be traceable back to what the machine does, with no hidden steps between the source and the hardware.

-----

## 7. Syntax

### 6.1 Dual Syntax Modes

Nyx supports two syntax styles that can be freely mixed within the same file. Both compile to identical AST nodes — there is no performance or semantic difference between them.

**Modern style** (Rust-inspired, type-annotated):

```nyx
fn add(a: i32, b: i32) -> i32 {
    x: i32 = a + b;
    return x;
}
```

**Classic style** (C-inspired, familiar to C developers):

```nyx
int add(int a, int b) {
    int x = a + b;
    return x;
}
```

**Mixed in the same file:**

```nyx
// Classic struct declaration
struct Point {
    int x;
    int y;
}

// Modern function using the classic struct
fn distance(a: Point, b: Point) -> f64 {
    dx: i32 = a.x - b.x;
    dy: i32 = a.y - b.y;
    return sqrt((dx * dx + dy * dy) as f64);
}
```

The parser resolves the mode per-declaration based on the opening token. No pragma or compiler flag is required.

### 6.2 Primitives

|Type |Size   |Notes                   |
|-----|-------|------------------------|
|i8   |1 byte |Signed integer          |
|i16  |2 bytes|Signed integer          |
|i32  |4 bytes|Signed integer (default)|
|i64  |8 bytes|Signed integer          |
|u8   |1 byte |Unsigned integer        |
|u16  |2 bytes|Unsigned integer        |
|u32  |4 bytes|Unsigned integer        |
|u64  |8 bytes|Unsigned integer        |
|f32  |4 bytes|IEEE 754 float          |
|f64  |8 bytes|IEEE 754 double         |
|bool |1 byte |true / false            |
|char |1 byte |ASCII character         |
|usize|arch   |Pointer-sized unsigned  |
|*T   |arch   |Raw pointer to T        |

### 6.3 Variables

```nyx
// Modern
x: i32 = 42;
name: *char = "nyx";

// Classic
int x = 42;
char* name = "nyx";

// Type inference (modern only)
let x = 42;        // inferred as i32
let y = 3.14;      // inferred as f64
```

### 6.4 Functions

```nyx
// Modern
fn multiply(a: i32, b: i32) -> i32 {
    return a * b;
}

// Classic
int multiply(int a, int b) {
    return a * b;
}

// Void return
fn log(msg: *char) {
    puts(msg);
}
```

### 6.5 Structs

```nyx
// Modern
struct Vec2 {
    x: f32,
    y: f32,
}

// Classic
struct Vec2 {
    float x;
    float y;
}

// Instantiation (both modes)
v: Vec2 = Vec2 { x: 1.0, y: 2.0 };
```

### 6.6 Enums

```nyx
enum Status {
    Ok,
    Err,
    Pending,
}

s: Status = Status::Ok;
```

### 6.7 Control Flow

```nyx
// if / else
if x > 0 {
    // ...
} else if x == 0 {
    // ...
} else {
    // ...
}

// while
while running {
    tick();
}

// for (range)
for i in 0..10 {
    print(i);
}

// for (C-style classic mode)
for (int i = 0; i < 10; i++) {
    print(i);
}

// loop (infinite)
loop {
    if done { break; }
}

// match
match status {
    Status::Ok      => handle_ok(),
    Status::Err     => handle_err(),
    Status::Pending => wait(),
}
```

-----

## 8. Memory Model

### 7.1 Philosophy

Nyx uses a **hybrid memory model**: manual allocation is the default, and ownership semantics are available as an opt-in layer for scopes or variables where safety is desired.

This means:

- You can write C-style code with full manual control.
- You can annotate specific variables or blocks with ownership semantics for automatic safety checks.
- The two modes coexist. Unsafe code must be explicitly marked.

### 7.2 Manual Allocation (Default)

```nyx
// Allocation and deallocation — classic
int* buf = alloc(1024 * sizeof(int));
free(buf);

// Modern syntax
buf: *i32 = alloc(1024 * sizeof(i32));
free(buf);
```

The compiler emits no implicit allocations. Every allocation is visible in source.

### 7.3 Ownership Mode (Opt-in)

Ownership can be applied at the variable level using the `own` keyword. An owned variable:

- Is automatically freed when it goes out of scope.
- Cannot be used after it has been moved or freed.
- Can be explicitly transferred with `move`.

```nyx
own buf: *i32 = alloc(256 * sizeof(i32));
// buf is freed automatically at end of scope

own a: *i32 = alloc(64 * sizeof(i32));
own b: *i32 = move(a);   // ownership transferred to b
// a is no longer valid — compiler error if accessed
// b is freed at end of scope
```

The ownership rules are enforced at compile time. There is no runtime overhead — the compiler inserts the `free` call at the appropriate scope exit.

### 7.4 Unsafe Blocks

Raw pointer arithmetic, casting, and dereferencing outside of owned variables requires an `unsafe` block.

```nyx
unsafe {
    ptr: *u8 = 0xB8000 as *u8;   // VGA text buffer
    *ptr = 'H' as u8;
}
```

The `unsafe` keyword can also annotate functions:

```nyx
unsafe fn write_port(port: u16, val: u8) {
    // inline assembly or raw I/O
}
```

-----

## 9. Error Model

### 8.1 Philosophy

Errors in Nyx are loud, informative, and (by default) fatal. There is no Result type, no exception hierarchy, and no silent failure. When something goes wrong, Nyx tells you exactly what happened, where, and why — then stops.

This is inspired by Python’s traceback model, adapted for a systems language.

### 8.2 Panic

A panic is a fatal runtime error. It prints a full diagnostic report and terminates the process.

```
[NYX PANIC] null pointer dereference
  → file: kernel/mem.nyx
  → line: 84, column: 12
  → function: alloc_page()
  → call stack:
      #0  alloc_page()         kernel/mem.nyx:84
      #1  map_virtual()        kernel/vmm.nyx:210
      #2  init_process()       kernel/proc.nyx:55
      #3  kmain()              kernel/main.nyx:12

  → cause: attempted to dereference pointer at address 0x0
  → hint: check that the allocator was initialized before calling alloc_page()
  → docs: https://nyx-lang.org/errors/E0042
```

Each panic type has a unique error code (`E0042`) with a corresponding documentation page explaining the cause, common patterns that trigger it, and how to fix it.

### 8.3 Panic Sources

The following trigger a panic by default:

- Null pointer dereference (outside `unsafe`)
- Array index out of bounds
- Integer overflow (debug builds; wraps in release)
- Stack overflow
- Explicit `panic("message")` call
- Division by zero

### 8.4 Recoverable Errors via `catch`

Panics can optionally be caught for non-fatal recovery in higher-level code:

```nyx
catch {
    result: i32 = risky_operation();
} on_panic(err) {
    log(err.message);
    return -1;
}
```

`catch` blocks are only valid in safe (non-bare-metal) mode. In `#[bare_metal]` mode, all panics are fatal by design — there is no recovery layer.

### 8.5 Error Codes and Documentation

Every compiler error and runtime panic has a unique code (e.g., `E0001`–`E9999`). Each code maps to an official documentation entry with:

- A plain-English explanation of what went wrong.
- The exact condition that triggered it.
- Common patterns that cause it.
- A fix recommendation.
- Link to related GitHub issues where applicable.

-----

## 10. Bare Metal Mode

### 9.1 Overview

Nyx supports a `#[bare_metal]` mode for kernel and embedded targets where no runtime is available — no stack unwinding, no panic handler, no allocator, no OS.

```nyx
#[bare_metal]

// No stdlib. No runtime. Raw hardware.
fn kmain() {
    unsafe {
        // Write directly to VGA text buffer
        ptr: *u8 = 0xB8000 as *u8;
        *ptr = 'N' as u8;
        *(ptr + 1) = 0x0F as u8;
    }
    loop {}
}
```

### 9.2 Bare Metal Guarantees

When `#[bare_metal]` is active:

- No standard library is linked.
- No runtime initialization code is emitted.
- No implicit stack frames for panic handling.
- The entry point is the raw function — no wrapper.
- The compiler does not insert any code the programmer did not write.
- `catch` blocks are a compile error — panics are always fatal.

### 9.3 Linker Control

Bare metal builds accept a linker script directly:

```
nyxc --bare-metal --linker kernel.ld -o kernel.bin kernel/main.nyx
```

-----

## 11. C Interoperability

### 10.1 Calling C from Nyx

```nyx
extern "C" {
    fn printf(fmt: *char, ...) -> i32;
    fn malloc(size: usize) -> *void;
    fn free(ptr: *void);
}

fn main() {
    printf("Hello from Nyx\n");
}
```

### 10.2 Calling Nyx from C

```nyx
#[export]
fn nyx_add(a: i32, b: i32) -> i32 {
    return a + b;
}
```

This exports with C linkage and no name mangling, callable from any C program.

### 10.3 C Header Generation

The compiler can emit a `.h` file for any `#[export]` functions:

```
nyxc --emit-header mylib.h mylib.nyx
```

-----

## 12. Modules

```nyx
// math.nyx
mod math {
    pub fn square(x: i32) -> i32 {
        return x * x;
    }
}

// main.nyx
import math;

fn main() {
    n: i32 = math::square(5);
}
```

Modules map to files. No circular imports. Visibility is `pub` (public) or private (default).

-----

## 13. Compiler Architecture

### 12.1 Pipeline

```
Source (.nyx)
    ↓
Lexer         → Token stream
    ↓
Parser        → AST (handles both syntax modes)
    ↓
Semantic Pass → Type checking, ownership validation, unsafe verification
    ↓
IR (Nyx IR)   → Flat, typed intermediate representation
    ↓
x86-64 Codegen → AT&T / Intel assembly
    ↓
Assembler     → Object file (.o)
    ↓
Linker        → ELF / PE / raw binary
```

### 12.2 Backend: x86-64 Assembly

The initial backend targets x86-64 and emits assembly directly — no LLVM, no C transpilation. This keeps the compiler lean, fast, and self-contained.

The assembler and linker are either bundled (for full self-hosting) or delegated to `nasm`/`ld` in the bootstrapping phase.

### 12.3 Self-Hosting Roadmap

|Phase|Description                                       |
|-----|--------------------------------------------------|
|0    |Bootstrap compiler written in C                   |
|1    |Lexer + parser written in Nyx, compiled by Phase 0|
|2    |Full compiler pipeline in Nyx                     |
|3    |Compiler compiles itself (self-hosting)           |
|4    |Bootstrap C compiler retired                      |

-----

## 14. The `nyx` Build Tool

Nyx ships with a first-class build tool called `nyx` — the equivalent of Cargo for the Nyx ecosystem. The goal is the same thing that makes Cargo great: one tool handles everything so you never have to think about build scripts, flags, or project wiring.

### 14.1 Project Structure

```
my_project/
├── nyx.toml          ← project manifest
├── src/
│   └── main.nyx      ← entry point
├── tests/
│   └── main_test.nyx ← test files
└── target/
    ├── debug/        ← debug builds
    └── release/      ← optimized builds
```

### 14.2 `nyx.toml` — Project Manifest

```toml
[project]
name    = "my_project"
version = "0.1.0"
entry   = "src/main.nyx"

[build]
target  = "x86-64"         # x86-64 | bare-metal | wasm (future)
linker  = ""               # optional linker script path

[dependencies]
libnyx = "0.2.0"           # future package registry

[profile.debug]
optimize    = false
debug_info  = true
overflow_check = true      # panics on integer overflow

[profile.release]
optimize    = true         # full x86-64 optimizations
debug_info  = false
overflow_check = false     # wraps silently for performance
strip       = true         # strip symbols from binary
```

### 14.3 Commands

|Command              |What it does                                      |
|---------------------|--------------------------------------------------|
|`nyx new <name>`     |Create a new project with the default structure   |
|`nyx build`          |Compile in debug mode → `target/debug/`           |
|`nyx build --release`|Compile with optimizations → `target/release/`    |
|`nyx run`            |Build (debug) and immediately execute             |
|`nyx run --release`  |Build (release) and immediately execute           |
|`nyx check`          |Type-check and validate without producing a binary|
|`nyx test`           |Build and run all tests in `tests/`               |
|`nyx test <name>`    |Run a single named test                           |
|`nyx clean`          |Delete the `target/` directory                    |
|`nyx emit-asm`       |Output x86-64 assembly alongside the binary       |
|`nyx emit-header`    |Generate a `.h` file for C interop                |

### 14.4 `nyx new`

```
$ nyx new hello_world
  created: hello_world/
  created: hello_world/nyx.toml
  created: hello_world/src/main.nyx

$ cat hello_world/src/main.nyx
fn main() {
    puts("Hello, world!\n");
}
```

### 14.5 Debug vs Release

**Debug** (`nyx build`) is fast to compile and maximally informative:

- No optimizations.
- Full panic diagnostics with source locations and call stacks.
- Integer overflow triggers a panic with error code and line number.
- Debug symbols included for use with a debugger.
- Output in `target/debug/<name>`.

**Release** (`nyx build --release`) is for shipping:

- Full x86-64 optimizations: inlining, dead code elimination, register allocation.
- Symbols stripped from the binary.
- Integer overflow wraps silently (matches C behaviour).
- Panic messages still printed, but without debug symbols.
- Output in `target/release/<name>`.

The difference between debug and release binaries should be significant — similar to the gap Rust achieves with Cargo.

### 14.6 Testing

Tests live in `tests/` and are plain Nyx functions annotated with `#[test]`:

```nyx
// tests/math_test.nyx
import math;

#[test]
fn test_square() {
    assert(math::square(4) == 16);
    assert(math::square(0) == 0);
    assert(math::square(-3) == 9);
}

#[test]
fn test_negative() {
    assert(math::square(-5) == 25);
}
```

Running `nyx test` output:

```
running 2 tests in tests/math_test.nyx

  ✓  test_square     (0.3ms)
  ✓  test_negative   (0.1ms)

result: 2 passed, 0 failed
```

A failed test:

```
running 2 tests in tests/math_test.nyx

  ✓  test_square     (0.3ms)
  ✗  test_negative   (0.1ms)

  [FAIL] test_negative
  → file: tests/math_test.nyx
  → line: 14, column: 5
  → assert(math::square(-5) == 24) failed
      left:  25
      right: 24
  → docs: https://nyx-lang.org/errors/E0101

result: 1 passed, 1 failed
```

### 14.7 Bare Metal Projects

For kernel targets, set `target = "bare-metal"` in `nyx.toml`:

```toml
[project]
name  = "mykernel"
entry = "src/kmain.nyx"

[build]
target = "bare-metal"
linker = "kernel.ld"
```

`nyx build` for a bare-metal project emits a raw binary with no runtime, linked according to the provided linker script. The `run` command is disabled for bare-metal targets — you boot it yourself.

-----

## 15. MVP Scope

The Minimum Viable Product covers what is required to write real programs and begin the self-hosting path.

**Compiler:**

- Lexer for both syntax modes
- Parser to unified AST
- Basic type checker
- Ownership checker for `own` variables
- x86-64 code generation
- Basic linker integration (ELF output)

**Language features:**

- Primitives, variables, functions
- Structs and enums
- Pointers and unsafe blocks
- `own` keyword and ownership enforcement
- Control flow: if/else, while, for, loop, match
- Modules and imports
- C extern declarations
- `#[bare_metal]` mode
- Panic with diagnostics

**Toolchain (`nyx` build tool):**

- `nyx new` — project scaffolding
- `nyx build` — debug builds to `target/debug/`
- `nyx build --release` — optimized builds to `target/release/`
- `nyx run` — build + execute in one step
- `nyx check` — validate without binary output
- `nyx test` — run annotated `#[test]` functions
- `nyx clean` — clear build artifacts
- `nyx emit-asm` — output x86-64 assembly
- `nyx emit-header` — generate `.h` for C interop
- `nyx.toml` manifest with debug/release profiles

**Not in MVP:**

- Generics / templates
- Closures
- Async / concurrency primitives
- Standard library beyond bare essentials
- Package manager
- IDE language server

-----

## 16. Success Criteria

Nyx is succeeding when:

1. A developer can write a bootloader in Nyx with `#[bare_metal]` and boot real hardware.
1. The Nyx compiler can compile itself (self-hosting achieved).
1. A CLI tool written in Nyx is indistinguishable in binary size and startup time from the same tool written in C.
1. A developer coming from C can read Nyx code without learning a new mental model.
1. Error messages are specific enough that a developer can fix a bug without searching the internet.

-----

## 17. Open Questions

|#|Question                                                      |Status       |
|-|--------------------------------------------------------------|-------------|
|1|Inline assembly syntax — AT&T style or custom `asm!` block?   |Undecided    |
|2|Generics strategy post-MVP — monomorphization or type erasure?|Undecided    |
|3|String type — null-terminated only, or a fat pointer `str`?   |Undecided    |
|4|Standard calling convention — System V ABI or custom?         |Lean System V|
|5|Debug info format — DWARF, or custom for self-hosted debugger?|Undecided    |
|6|Package manager design and registry                           |Post-MVP     |

-----

*End of PRD v0.3*
