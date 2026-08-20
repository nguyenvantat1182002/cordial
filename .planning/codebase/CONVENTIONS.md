# Coding Conventions

**Analysis Date:** 2026-08-20

## Naming Patterns

**Files:**
- Rust modules: `snake_case.rs` (e.g. `client_settings.rs`, `host_window.rs`, `pipewire_backend.rs`)
- C++ source & headers: `snake_case.cpp`, `snake_case.h` (e.g. `android_classes.cpp`, `pipewire_backend.h`)
- Markdown documentation: `kebab-case.md` for topics, `ADR-NNN-kebab-case.md` for architecture records, `UPPERCASE.md` for root standards (`AGENTS.md`, `README.md`, `CONTRIBUTING.md`)

**Functions:**
- Rust: `snake_case` (e.g. `pass_key_event`, `find_symbol`, `wire_refresh_rate`)
- C++: `camelCase` or `snake_case` matching Android/JNI conventions (e.g. `fill_pcm`, `registerNatives`, `Java_com_roblox_client_GameActivity_nativeOnCreate`)

**Variables:**
- Rust: `snake_case` for local variables and fields; `SCREAMING_SNAKE_CASE` for constants and statics (e.g. `PATH_MAX`, `DEFAULT_TIMEOUT`)
- C++: `snake_case` for locals and members; `SCREAMING_SNAKE_CASE` for macros and compile-time constants

**Types:**
- Rust: `PascalCase` for structs, enums, and traits (e.g. `WaylandWindow`, `HostWindowCell`, `ContentHash`)
- C++: `PascalCase` or standard C types (e.g. `PipewireBackend`, `JNINativeMethod`)

## Code Style

**Formatting:**
- Rust: `rustfmt` standard style (4 spaces indentation, 100 max line width)
- C++: `clang-format` based on LLVM/Google style with 4 spaces indentation

**Linting:**
- Rust: `cargo clippy` with workspace-level lints; minimal use of `unsafe` with documented safety invariants
- C++: Clang compiler diagnostics with strict warnings (`-Wall`, `-Wextra`)

## Import Organization

**Order:**
1. Standard library imports (`std::path::PathBuf`, `std::sync::Arc`)
2. External crate dependencies (`gtk4::prelude::*`, `zbus::blocking::Connection`, `serde::Deserialize`)
3. Workspace crate dependencies (`cordial_plugins::manifest::PluginManifest`)
4. Internal crate modules (`crate::android::input`, `super::profile`)

**Path Aliases & Re-exports:**
- Clean module hierarchies using `pub use` only at crate boundaries (`crates/cordial-shell/src/lib.rs`)

## Error Handling

**Patterns:**
- Rust: Return explicit `Result<T, E>` types; avoid `unwrap()` or `expect()` in runtime paths unless an invariant has been strictly verified.
- C++ & JNI: Never throw C++ exceptions across FFI boundaries; return explicit error codes (`SL_RESULT_FEATURE_UNSUPPORTED`, `NULL`, `-1`).
- **Never Fake Success:** Stubs must return honest failure codes rather than spoofing success, keeping failure visible and traceable.

## Logging

**Framework:**
- Android `liblog` interceptor (`native/liblog.cpp`) routing engine logs to stdout
- Rust `eprintln!` and structured console logging for runtime diagnostics

**Patterns:**
- Prefix log messages with subsystem identifiers (e.g. `[cordial-runtime]`, `[DFLog::...]`, `[Wayland]`).
- Detailed diagnostics gated behind environment variables (`CORDIAL_TRACE_TEXT`, `CORDIAL_TRACE_PATHS`, `CORDIAL_JNI_TRACE`).

## Comments

**When to Comment:**
- **Anchor in Failure:** Comments must explain *why* code exists, referencing the specific bug or regression that motivated the design.
- **British-ish Prose:** Consistent, professional tone; no emojis in code or comments; no bullet-list comment blocks in source files.
- **Label Inferences:** If a behavior cannot be directly observed and verified, label it `INFERRED` in comments.

**Docstrings:**
- Rustdoc `///` on public structs, functions, and traits detailing safety requirements and invariants.

## Function Design

**Size:**
- Modular functions focused on a single responsibility; complex state transitions decomposed into helper methods.

**Parameters:**
- Borrowed references (`&str`, `&Path`, `&[u8]`) preferred over owned types for input arguments.

**Return Values:**
- Strongly typed enums or `Result` types representing all possible outcome states.

## Module Design

**Exports:**
- Minimal public surface area; internal implementation details kept private to crate modules.

**Barrel Files:**
- `mod.rs` or `<crate>/src/lib.rs` exports primary entry types and traits cleanly.

---

*Convention analysis: 2026-08-20*
