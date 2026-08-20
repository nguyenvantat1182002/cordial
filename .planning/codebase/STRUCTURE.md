# Codebase Structure

**Analysis Date:** 2026-08-20

## Directory Layout

```
cordial/
├── .agents/                 # GSD configuration, workflows, skills, and agent manifests
├── .github/workflows/       # GitHub Actions CI/CD workflows (Flatpak build pipeline)
├── crates/                  # Rust workspace members
│   ├── cordial-linker-sys/  # FFI bindings to AOSP bionic dynamic linker & C++ shims
│   ├── cordial-plugins/     # Plugin engine, capability broker, sandboxing, marketplace
│   ├── cordial-runtime/     # Core runtime loader, symbol table, Android framework shims
│   ├── cordial-shell/       # GTK4/libadwaita launcher UI, settings, and profile manager
│   └── cordial-update/      # APK download, validation, engine caching, changelog parser
├── docs/                    # Architecture records, analysis data, traces, and guides
│   ├── adr/                 # Architecture Decision Records (ADR-001 through ADR-018)
│   ├── analysis/            # Symbol tables, JNI inventory, FastFlag data, render gate
│   ├── design/              # Deep-dive design notes and subsystem specifications
│   └── traces/              # Real Android logcat reference captures
├── native/                  # C/C++ Android framework shims, JNI virtual machine, PipeWire
├── packaging/               # Flatpak manifest, desktop files, AppStream metadata, icons
├── patches/                 # Submodule and external compatibility patches
├── plugins/                 # First-party plugins (discord-presence, flag-inspector)
├── third_party/             # Vendored submodules (libjnivm, mcpelauncher-linker, libbadcpu)
├── tools/                   # Python/Shell diagnostic and analysis utilities
├── Cargo.toml               # Root workspace Cargo manifest
├── flake.nix                # Nix Flake development shell definition
└── justfile                 # Just task runner recipe definitions
```

## Directory Purposes

**`crates/`:**
- Purpose: Contains all Rust workspace crates implementing the host application and core runtime.
- Contains: Rust source files (`src/`), manifests (`Cargo.toml`), and build scripts (`build.rs`).
- Key files: `crates/cordial-runtime/src/lib.rs`, `crates/cordial-shell/src/main.rs`.

**`native/`:**
- Purpose: Houses C++ implementations of Android framework APIs, JNI bindings, and audio backends.
- Contains: C++ source and header files, CMake build definitions (`CMakeLists.txt`).
- Key files: `native/android_classes.cpp`, `native/opensles.cpp`, `native/pipewire_backend.cpp`, `native/liblog.cpp`.

**`docs/`:**
- Purpose: Comprehensive project documentation, historical findings, decision records, and Android traces.
- Contains: Markdown documents, TSV symbol tables, logcat trace dumps.
- Key files: `docs/HANDOVER.md`, `docs/NEXT.md`, `docs/architecture.md`, `docs/findings.md`.

**`docs/adr/`:**
- Purpose: Immutable record of architectural decisions and trade-offs.
- Contains: Numbered ADR Markdown files (`ADR-001` to `ADR-018`).
- Key files: `docs/adr/ADR-001-in-process-hooking.md`, `docs/adr/ADR-011-wayland-and-libadwaita.md`.

**`plugins/`:**
- Purpose: First-party and reference plugin implementations written in TypeScript for Deno.
- Contains: Plugin manifests (`manifest.json`), TypeScript code (`index.ts`).
- Key files: `plugins/discord-presence/index.ts`, `plugins/flag-inspector/index.ts`.

**`packaging/`:**
- Purpose: Package definitions for distribution on Linux platforms (Flatpak, Flathub).
- Contains: Flatpak manifest (`io.github.luohoa97.Cordial.yml`), AppStream metadata, desktop shortcuts, application icons.

**`third_party/`:**
- Purpose: Vendored upstream submodules required for binary loading and emulation.
- Contains: `mcpelauncher-linker` (AOSP Bionic linker port), `libjnivm` (JNI VM), `libbadcpu` (x86 CPU emulation).

**`tools/`:**
- Purpose: Offline analysis scripts for investigating DEX signatures, symbol diffs, and test runners.
- Contains: Python scripts, shell scripts.
- Key files: `tools/dex_signature.py`, `tools/engine-text-diff.py`, `tools/join-run.sh`.

## Key File Locations

**Entry Points:**
- `crates/cordial-shell/src/main.rs`: Desktop launcher application binary (`cordial-shell`).
- `crates/cordial-runtime/src/bin/load.rs`: Standalone engine runner binary (`cordial-run`).
- `plugins/discord-presence/index.ts`: Entry point for the Discord RPC plugin.

**Configuration:**
- `Cargo.toml`: Workspace configuration and dependency versions.
- `native/CMakeLists.txt`: CMake build configuration for native shims and library linking.
- `justfile`: Common commands for building, running, testing, and debugging.
- `flake.nix`: Nix development environment definition.

**Core Logic:**
- `crates/cordial-runtime/src/symtab.rs`: Virtual symbol table mapping ~650 Android symbols.
- `crates/cordial-runtime/src/android/wayland.rs`: Wayland display protocol integration and subsurface binding.
- `crates/cordial-runtime/src/android/input.rs`: Input translation from Wayland to Android `MotionEvent`/`KeyEvent`.
- `crates/cordial-plugins/src/broker.rs`: Capability validation and dispatch engine for plugins.
- `native/pipewire_backend.cpp`: Low-latency audio backend feeding PipeWire streams.

**Testing:**
- `native/pipewire_backend_test.cpp`: Native unit test for PipeWire audio buffer handling.
- `crates/cordial-update/src/download.rs`: Unit tests for APK download chunking and validation.
- `crates/cordial-plugins/src/sign.rs`: Cryptographic verification unit tests.

## Naming Conventions

**Files:**
- Rust: `snake_case.rs` (e.g. `client_settings.rs`, `host_window.rs`)
- C++: `snake_case.cpp` and `snake_case.h` (e.g. `pipewire_backend.cpp`, `android_classes.cpp`)
- Documentation: `kebab-case.md` or `UPPERCASE.md` (e.g. `framework-api-inventory.md`, `README.md`)
- ADRs: `ADR-NNN-kebab-case.md` (e.g. `ADR-011-wayland-and-libadwaita.md`)

**Directories:**
- Crates: `kebab-case` prefixed with `cordial-` (e.g. `cordial-runtime`, `cordial-plugins`)
- Subdirectories: `snake_case` or `kebab-case` (e.g. `crates/cordial-runtime/src/android/`)

## Where to Add New Code

**New Android Framework Class or JNI Method:**
- Implementation: Add C++ binding in `native/android_classes.cpp` or a new module in `native/` registered in `native/CMakeLists.txt`.
- Registration: Declare the Java class/method in `native/jni_shim.cpp` or `crates/cordial-runtime/src/stubs.rs`.
- Reference: Verify method signature against `tools/dex_signature.py` and `docs/traces/`.

**New Bionic / Libc Symbol:**
- Implementation: Add host forwarding or Rust emulation in `crates/cordial-runtime/src/bionic/`.
- Routing: Register symbol and soname in `crates/cordial-runtime/src/symtab.rs`.
- Documentation: Update `docs/analysis/undefined-symbols.tsv`.

**New Plugin Capability:**
- Definition: Add capability variant and request/response types in `crates/cordial-plugins/src/capability.rs`.
- Broker Handler: Implement execution logic in `crates/cordial-plugins/src/broker.rs`.
- Grant Check: Update permission rules in `crates/cordial-plugins/src/grants.rs`.

**New Launcher UI Component / Setting:**
- View/Dialog: Add GTK4/libadwaita component in `crates/cordial-shell/src/settings.rs` or new module in `crates/cordial-shell/src/`.
- Configuration Persistence: Update `crates/cordial-shell/src/shell_config.rs`.

**New Unit / Integration Test:**
- Rust Tests: Add `#[cfg(test)] mod tests` block at the bottom of the corresponding module or under `crates/<crate>/tests/`.
- Native C++ Tests: Add test executable in `native/` and register in `native/CMakeLists.txt`.

## Special Directories

**`.planning/`:**
- Purpose: GSD workflow planning documents, roadmaps, and codebase maps.
- Generated: Semi-automated via GSD commands.
- Committed: Yes.

**`docs/traces/`:**
- Purpose: Pristine logcat recordings from real Android devices for ground-truth behavior comparison.
- Generated: Captured manually on test hardware.
- Committed: Yes.

**`third_party/`:**
- Purpose: Git submodules for external C++ libraries.
- Generated: Cloned via `git submodule update --init --recursive`.
- Committed: Tracked as git submodules.

---

*Structure analysis: 2026-08-20*
