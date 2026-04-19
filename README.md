# Nyx
PRD: Nyx

Overview

Nyx is a low-level, statically typed systems programming language inspired by C and Rust. It is designed to give direct control over memory and performance like C, while adding safety defaults and cleaner abstractions like Rust.

Goals
	•	Provide C-like performance and control.
	•	Add safety by default without making the language feel heavy.
	•	Keep the language small, explicit, and predictable.
	•	Make it practical for systems programming, embedded work, and performance-critical software.
	•	Support seamless C interoperability.

Core Design Principles
	•	Explicit over implicit.
	•	Safe by default.
	•	Unsafe code must be clearly marked.
	•	Minimal runtime.
	•	Fast compilation.
	•	Predictable behavior.

Key Features
	•	Static typing with type inference where obvious.
	•	Manual and controlled memory access.
	•	Ownership/safety model simpler than Rust.
	•	unsafe blocks for raw pointer and low-level operations.
	•	Strong C interop.
	•	Functions, structs, enums, and modules as core language features.
	•	No unnecessary language magic.

Syntax Direction
	•	Clean, compact, C-like syntax.
	•	Readable but not verbose.
	•	Should feel sharp and serious, not like a scripting language.

MVP Scope
	•	Lexer, parser, AST.
	•	Basic type system.
	•	Variables, functions, structs, control flow.
	•	Pointers and unsafe blocks.
	•	Simple compiler or interpreter backend.
	•	C interop for basic function calls.

Non-Goals
	•	Not a Python/Ruby-style scripting language.
	•	Not a Rust clone.
	•	Not a garbage-collected language.
	•	Not a huge ecosystem-first language at the start.

Success Criteria

Nyx should feel like a language that:
	•	lets experienced developers stay close to the machine,
	•	reduces accidental memory bugs compared to C,
	•	stays simple enough to learn and build with,
	•	and is distinct enough to have its own identity.

