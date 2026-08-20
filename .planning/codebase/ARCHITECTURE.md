<!-- refreshed: 2026-08-20 -->
# Architecture

**Analysis Date:** 2026-08-20

## System Overview

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Your Linux Desktop                                │
├──────────────────────────────────────┬──────────────────────────────────────┤
│   cordial-shell (Launcher UI)        │   cordial-plugins (Plugin Engine)    │
│   GTK4 · libadwaita · profiles       │   Deno runtime · bwrap sandbox       │
│   `crates/cordial-shell`             │   `crates/cordial-plugins`           │
└──────────────────┬───────────────────┴──────────────────┬───────────────────┘
                   │ spawns with arguments                │ IPC capability
                   │ & hands over profile flock           │ broker protocol
                   ▼                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│               cordial-run (One Process = One Instance = One Window)         │
├─────────────────────────────────────────────────────────────────────────────┤
│   Framework Layer (GameActivity, Input, Window, Asset, Accessibility, Audio) │
│   `crates/cordial-runtime/src/android/` · `native/`                         │
├──────────────────────────────────────┬──────────────────────────────────────┤
│   Symbol Routing Table (symtab.rs)   │   libjnivm (Java Virtual Machine)    │
│   ~650 symbols: Cordial / Glibc / Stub│   Answering 518 static JNI natives  │
│   `crates/cordial-runtime/src/symtab`│   `third_party/libjnivm`             │
├──────────────────────────────────────┴──────────────────────────────────────┤
│   Ported AOSP Bionic Linker (mcpelauncher-linker)                           │
│   Maps ELF segments, walks DT_NEEDED, applies relocations                    │
│   `third_party/mcpelauncher-linker` · `crates/cordial-linker-sys`            │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ dlopen / JNI_OnLoad
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   libroblox.so (Roblox Official Android x86-64 Engine — User Supplied)      │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │ System interactions mediated outward
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│   Host Subsystems: Wayland (xdg-shell) · Vulkan/GLES · PipeWire · D-Bus     │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| `cordial-shell` | Desktop launcher, profile switcher, settings UI, update manager, deep link handler | `crates/cordial-shell/src/lib.rs` |
| `cordial-runtime` | Core runtime orchestration, symbol resolution, Android system services emulation, Wayland surface binding | `crates/cordial-runtime/src/lib.rs` |
| `cordial-plugins` | Plugin lifecycle management, Deno execution, Bubblewrap sandboxing, brokered IPC capability handling | `crates/cordial-plugins/src/lib.rs` |
| `cordial-update` | Downloading and unpacking official APKs, caching shared libraries, checking version diffs and release notes | `crates/cordial-update/src/lib.rs` |
| `cordial-linker-sys` | Rust FFI bindings to the ported AOSP bionic dynamic linker and C++ runtime shims | `crates/cordial-linker-sys/src/lib.rs` |
| `native/` | C++ Android framework implementations, JNI method registrations, OpenSL ES audio bridge, and logger interception | `native/CMakeLists.txt` |
| `native/pipewire_backend.cpp` | Low-latency audio backend consuming PCM stream from OpenSL ES and feeding PipeWire | `native/pipewire_backend.cpp` |
| `crates/cordial-runtime/src/android/wayland.rs` | Hand-marshalled Wayland protocol implementation, subsurface attachment, text input v3, cursor locking | `crates/cordial-runtime/src/android/wayland.rs` |

## Pattern Overview

**Overall:** Non-Invasive Foreign Library Runtime & Mediated Platform Emulation.

**Key Characteristics:**
- **Outward-Only Mediation:** `libroblox.so` is loaded, relocated, and invoked without binary modification, memory patching, or API hooking (ADR-001). Every arrow points outward — the engine queries, Cordial answers.
- **Brokered Plugin Capabilities:** Plugins run as isolated subprocesses under Deno and Bubblewrap (`bwrap`). They receive structured JSON payloads and invoke effects; they never receive raw file descriptors, network sockets, or D-Bus connections (ADR-007, ADR-018).
- **Single Instance per Profile (`flock`):** Storage isolation guarantees one process per profile directory to prevent database and cookie corruption (ADR-012).
- **Virtual Dynamic Library Synthesis:** Symbol resolution splits imported names into three deterministic tiers: Cordial custom shim, host glibc forwarding, or honest failure stubs.

## Layers

**Shell Layer (`crates/cordial-shell`):**
- Purpose: Provides user interface for profile management, FastFlag configuration, APK installation, and game launch.
- Location: `crates/cordial-shell/src/`
- Contains: GTK4/libadwaita dialogs, settings views, WebKit authentication windows, updater integration.
- Depends on: `cordial-plugins`, `cordial-update`, `gtk4`, `libadwaita`, `gdk4-wayland`.
- Used by: End users as desktop application entry point.

**Runtime Layer (`crates/cordial-runtime`):**
- Purpose: Sets up the execution environment, instantiates the virtual machine, wires Wayland and audio, and invokes the engine.
- Location: `crates/cordial-runtime/src/`
- Contains: Symbol resolution tables, Android Looper/GameActivity bridge, input translation, cookies, client settings.
- Depends on: `cordial-linker-sys`, `cordial-plugins`, `cordial-shell`, `gtk4`, `libadwaita`, `zbus`, `zip`, `libmimalloc-sys`.
- Used by: `cordial-run` binary spawned by `cordial-shell`.

**Linker & Native Shim Layer (`crates/cordial-linker-sys`, `native/`):**
- Purpose: Loads Android ELF binaries compiled against Bionic libc and resolves symbols to Linux host functions or shims.
- Location: `native/`, `third_party/mcpelauncher-linker/`, `third_party/libjnivm/`
- Contains: AOSP linker, libjnivm JNI emulation, Android native headers (`android/native_window.h`, `android/sensor.h`), OpenSL ES audio shim.
- Depends on: Clang runtime, PipeWire headers, system libc.
- Used by: `cordial-runtime`.

**Plugin Subsystem Layer (`crates/cordial-plugins`):**
- Purpose: Orchestrates sandboxed background plugins, mediates events, and executes safe host capabilities.
- Location: `crates/cordial-plugins/src/`, `plugins/`
- Contains: Capability broker, permissions evaluator, package unpacker, signature validator.
- Depends on: `minisign-verify`, `ed25519-dalek`, `zbus`, `tar`, `zstd`.
- Used by: `cordial-runtime` and `cordial-shell`.

## Data Flow

### Primary Request Path (Application Launch & Game Join)

1. **Launch Initiation:** User selects profile and launches game or clicks `roblox://` deep link (`crates/cordial-shell/src/launch.rs:1`).
2. **Profile Lock Acquisition:** Shell acquires `flock` on `$XDG_DATA_HOME/cordial/profiles/<profile>/` and passes the file descriptor to the child process (`crates/cordial-shell/src/profile.rs:1`).
3. **Child Execution:** `cordial-run` binary starts with `--lib-dir`, `--apk`, `--profile`, and `--game-activity` arguments (`crates/cordial-runtime/src/bin/load.rs:1`).
4. **Symbol Table Initialization:** `symtab.rs` constructs the 13 virtual Android libraries (`libc.so`, `libm.so`, `libGLESv2.so`, `libEGL.so`, `libandroid.so`, etc.) and registers ~650 symbols (`crates/cordial-runtime/src/symtab.rs:1`).
5. **Engine Loading:** Ported bionic linker loads `libroblox.so`, processes ELF relocations against the virtual symbol table, and initializes JNI bindings (`crates/cordial-linker-sys/src/lib.rs:1`).
6. **Native Window & Event Loop Creation:** Wayland subsurface is created inside GTK host window (`crates/cordial-runtime/src/android/wayland.rs:1`), `GameActivity_onCreate` is invoked, and `ANativeActivity` callbacks begin execution.
7. **Frame Rendering & Event Dispatch:** Engine renders frames via Vulkan/EGL to the Wayland subsurface while input events flow from Wayland to `android/input.rs`.

### Secondary Flow Name (Plugin Event & Capability Brokerage)

1. **Event Emission:** Engine logs or triggers milestone (e.g. `onJoin`, player load); `cordial-runtime` catches event via `liblog` or JNI interceptor (`native/liblog.cpp`).
2. **Broker Dispatch:** Event is serialized to JSON and broadcast to subscribed plugins over IPC pipe (`crates/cordial-plugins/src/events.rs:1`).
3. **Plugin Action:** Plugin script processes event and requests capability (e.g. `presence.set` with activity details) (`plugins/discord-presence/index.ts`).
4. **Capability Execution:** Host broker validates capability permissions against plugin manifest grants and performs effect (e.g. updates Discord UNIX socket) (`crates/cordial-plugins/src/broker.rs:1`).

**State Management:**
- Application state is partitioned strictly by profile.
- Configuration and FastFlags are persisted in JSON files in the profile root.
- In-memory state is maintained by actors and channels across the runtime event loop.

## Key Abstractions

**`Symtab` (Symbol Routing Table):**
- Purpose: Maps Android Bionic dynamic symbol requests to host glibc, Cordial shims, or stubs.
- Examples: `crates/cordial-runtime/src/symtab.rs`
- Pattern: Virtual Dynamic Linker Symbol Registry.

**`WaylandWindow` & `HostWindowCell`:**
- Purpose: Integrates the engine's Wayland subsurface with the GTK4/libadwaita top-level window.
- Examples: `crates/cordial-runtime/src/android/wayland.rs`, `crates/cordial-shell/src/host_window.rs`
- Pattern: Subsurface Composite Presenter.

**`Broker` & `Capability`:**
- Purpose: Exposes host operations to untrusted plugins via typed capability requests without exposing raw handles.
- Examples: `crates/cordial-plugins/src/broker.rs`, `crates/cordial-plugins/src/capability.rs`
- Pattern: Mediated Capability Security Broker.

**`Profile`:**
- Purpose: Encapsulates user storage directory, cookie jars, configuration, and file locks.
- Examples: `crates/cordial-runtime/src/profile.rs`, `crates/cordial-shell/src/profile.rs`
- Pattern: Isolated Workspace Entity with Advisory File Locking.

## Entry Points

**Desktop Launcher Binary:**
- Location: `crates/cordial-shell/src/main.rs` (`cordial-shell`)
- Triggers: Desktop application menu, CLI invocation, or protocol URL handler.
- Responsibilities: Displays UI, manages profiles, updates engine cache, launches `cordial-run`.

**Runtime Engine Runner Binary:**
- Location: `crates/cordial-runtime/src/bin/load.rs` (`cordial-run`)
- Triggers: Spawned by `cordial-shell` or run directly by developers.
- Responsibilities: Initializes linker, loads `libroblox.so`, runs GameActivity event loop.

**Plugin Subprocess:**
- Location: `plugins/*/index.ts` spawned via Deno CLI inside Bubblewrap sandbox.
- Triggers: Started by `cordial-plugins::host` on runtime startup.
- Responsibilities: Implements user-extended functionality.

## Architectural Constraints

- **No In-Process Hooking:** Strictly forbidden to inject scripts, patch memory, or hook Roblox functions (ADR-001).
- **No Roblox Assets or Code Vendored:** Zero proprietary assets or decompiled binaries committed in repository (AGENTS.md).
- **Compiler Compatibility:** C++ native components require Clang/LLVM due to AOSP Bionic C11 header syntax (`_Atomic`).
- **Wayland Connection Sharing:** Subsurfaces in Wayland must share the exact same `wl_display` connection as the parent GTK window (ADR-011).
- **Profile Exclusivity:** Maximum of one running instance per profile directory enforced via POSIX `flock` (ADR-012).

## Anti-Patterns

### Lying Stubs

**What happens:** Writing a stub that returns a fake success code (e.g. `0` or non-null dummy pointer) to bypass an unimplemented feature.
**Why it's wrong:** The engine proceeds assuming the capability exists and crashes later in an unrelated subsystem with unhelpful stack traces.
**Do this instead:** Return explicit error codes (e.g. `SL_RESULT_FEATURE_UNSUPPORTED` in `native/opensles.cpp` or `EOPNOTSUPP`).

### Inferring from Stripped Binaries Instead of Checking Traces

**What happens:** Disassembling stripped `libroblox.so` and guessing calling conventions or arguments.
**Why it's wrong:** High risk of incorrect assumptions (9 consecutive wrong conclusions were made this way in early development).
**Do this instead:** Grep `docs/traces/` logcat captures from real Android hardware and compare against working implementations in reference projects.

### Using Raw Present Counts as Frame Rate

**What happens:** Measuring `vkQueuePresentKHR` or frame presents over a wall-clock interval and reporting it as FPS.
**Why it's wrong:** Roblox engages an idle throttle after 13 seconds of inactivity, dropping presents to exactly 1.0/s.
**Do this instead:** Measure frame rate while continuously driving input and report input frequency alongside frame frequency.

## Error Handling

**Strategy:** Explicit Rust `Result<T, E>` types propagated with `?` operator; honest C/C++ status codes returned across FFI boundaries.

**Patterns:**
- Linker and loader errors abort early with detailed symbol diagnostics.
- Plugin errors isolate to the plugin process without crashing the core runtime.
- Network and CDN fetch failures fall back gracefully to cached configurations.

## Cross-Cutting Concerns

**Logging:** Android `liblog` routing to stdout with subsystem tags; JNI unresolved symbols flagged in red.
**Validation:** Cryptographic signature verification for all plugins (`minisign` / `ed25519`); JSON schema validation on configuration files.
**Authentication:** Sandboxed WebKitGTK authentication sheet with Freedesktop Secret Service backend.

---

*Architecture analysis: 2026-08-20*
