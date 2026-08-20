# Technology Stack

**Analysis Date:** 2026-08-20

## Languages

**Primary:**
- Rust 1.75+ (Edition 2021) - Core runtime, shell UI, plugin subsystem, update manager, linker bindings (`crates/cordial-runtime`, `crates/cordial-shell`, `crates/cordial-plugins`, `crates/cordial-update`, `crates/cordial-linker-sys`)
- C++17 (Clang required) - Android framework shims, JNI virtual machine bindings, PipeWire audio backend, AOSP bionic linker integration (`native/`, `third_party/libjnivm`, `third_party/mcpelauncher-linker`)
- C11 - Low-level libc/bionic bridging, dynamic linker hooks (`native/shim.cpp`, `native/legacy_stdio.cpp`)

**Secondary:**
- Python 3 - Analysis scripts, DEX method/signature extractors, Flatpak Cargo vendoring generator (`tools/dex_method.py`, `tools/dex_signature.py`, `packaging/cargo-sources.py`)
- TypeScript / JavaScript - Cordial plugins running on Deno runtime (`plugins/discord-presence`, `plugins/flag-inspector`)
- Shell / Bash / Just - Build automation, workflow orchestration, verification harnesses (`justfile`, `tools/join-run.sh`, `packaging/build-flatpak.sh`)
- Nix - Declarative development environment (`flake.nix`)

## Runtime

**Environment:**
- Linux x86_64 native host (direct execution without virtual machine or Android container)
- Display Server: Wayland native (`xdg-shell`, `zwp_text_input_v3`, `wl_subsurface`)
- Audio Server: PipeWire 0.3 (`libpipewire-0.3.so` dynamically loaded at runtime via `dlopen`)
- System Bus: D-Bus (Secret Service, NetworkManager, AT-SPI accessibility, GameMode)

**Package Manager:**
- Cargo (Rust package manager, workspace resolver v2)
  - Lockfile: `Cargo.lock` present and tracked
- CMake 3.16+ (Native build system for `native/` and `third_party/` submodules)

## Frameworks

**Core:**
- GTK 4 (`gtk4` crate v0.11 with feature `v4_20`) - Primary UI framework for launcher window and settings dialogs (`crates/cordial-shell/src/window.rs`)
- libadwaita (`libadwaita` crate v0.9 with feature `v1_8`) - GNOME HIG widgets, `AdwDialog`, `AdwToolbarView`, `AdwSwitchRow` (`crates/cordial-shell/src/settings.rs`)
- WebKitGTK 6.0 (`webkit6` crate v0.6 optional via `webview` feature) - In-app web views for authentication, account settings, and Robux purchases (`crates/cordial-shell/src/webview.rs`)
- libjnivm (`third_party/libjnivm`) - Replaces Android ART runtime by answering JNI calls from `libroblox.so`
- AOSP Bionic Linker (`third_party/mcpelauncher-linker`) - Custom ELF loader mapping Android shared objects on Linux

**Testing:**
- Built-in Rust Test Harness (`cargo test --workspace`) - Unit tests for update management, settings serialization, plugin verification, URI parsing
- Custom C++ Post-Build Test Harness (`native/pipewire_backend_test.cpp`) - Verifies PipeWire PCM ring buffer fill and underrun-to-silence handling without requiring a live audio daemon

**Build/Dev:**
- `cmake` (crate v0.1) - Invokes CMake from Rust build scripts (`crates/cordial-linker-sys/build.rs`)
- `just` - Command runner orchestrating containerized, host, and test runs (`justfile`)
- `pkg-config` / `pkgconf` - Detection of system libraries and headers (`native/CMakeLists.txt`)

## Key Dependencies

**Critical:**
- `zbus` (v5.18.0, features `["blocking"]`) - High-level D-Bus client for AT-SPI accessibility bridge, Secret Service cookie persistence, and NetworkManager metered connection detection (`crates/cordial-runtime/src/android/accessibility.rs`, `crates/cordial-runtime/src/secrets.rs`, `crates/cordial-update/src/metered.rs`)
- `gdk4-wayland` (v0.11) - Raw Wayland display and surface extraction (`wl_display`, `wl_surface`) to attach Android subsurface (`crates/cordial-shell/src/host_window.rs`)
- `libmimalloc-sys` (v0.1.49, feature `extended`) - Provides discoverable `libmimalloc.so` symbols required by Roblox engine's memory allocator detection (`crates/cordial-runtime/src/mimalloc_lib.rs`)
- `minisign-verify` (v0.2.5) & `ed25519-dalek` (v3.0.0) - Cryptographic signature verification for plugin packages and marketplace index (`crates/cordial-plugins/src/sign.rs`, `crates/cordial-plugins/src/marketplace.rs`)
- `zip` (v2, features `["deflate"]`) - Streaming APK archive parser for asset extraction and `libroblox.so` unpacking (`crates/cordial-runtime/src/android/asset.rs`, `crates/cordial-update/src/apk.rs`)

**Infrastructure:**
- `ureq` (v3.3.0) - Synchronous HTTP client for FastFlag CDN querying and official APK downloads (`crates/cordial-runtime/src/client_settings.rs`, `crates/cordial-update/src/download.rs`)
- `serde` (v1, features `["derive"]`) & `serde_json` (v1) - JSON serialization for configuration, state, capabilities, and Roblox client settings (`crates/cordial-shell/src/shell_config.rs`, `crates/cordial-plugins/src/protocol.rs`)
- `sha2` (v0.10) & `zstd` (v0.13) & `tar` (v0.4) - Hash integrity verification and plugin package extraction (`crates/cordial-plugins/src/unpack.rs`, `crates/cordial-update/src/sha256.rs`)
- `pulldown-cmark` (v0.13.4) - Markdown parsing for release notes and update summaries (`crates/cordial-update/src/changelog.rs`)

## Configuration

**Environment:**
- `CORDIAL_PROFILE_ROOT` - Overrides the default profile root directory for isolated test runs
- `CORDIAL_TRACE_TEXT` - Enables verbose tracing of IME/text-input events
- `CORDIAL_TRACE_PATHS` - Enables logging for path-taking libc calls
- `CORDIAL_JNI_TRACE` - Compile-time/runtime toggle for verbose JNI method call logging (disabled by default due to high overhead)
- `XDG_DATA_HOME` & `XDG_CONFIG_HOME` - Standard Freedesktop paths for profile storage, configuration, and engine cache

**Build:**
- `Cargo.toml` - Workspace configuration defining crate dependencies and release profile optimizations (`debug = true` enabled in release for diagnostics)
- `native/CMakeLists.txt` - C++ compiler flags (`-DPATH_MAX=4096`, Clang enforcement, PipeWire header detection)
- `packaging/io.github.luohoa97.Cordial.yml` - Flatpak manifest targeting `org.gnome.Platform` runtime 47
- `flake.nix` - Nix flake specifying LLVM/Clang, GTK4, libadwaita, and Wayland build dependencies

## Platform Requirements

**Development:**
- Operating System: Linux x86_64
- Compilers: Clang / Clang++ (GCC cannot compile AOSP Bionic C11 `_Atomic` headers), Rust 1.75+
- System Headers & Libraries: GTK4 (`gtk4-devel`), libadwaita (`libadwaita-devel`), WebKitGTK 6.0 (`webkitgtk6.0-devel` optional), PipeWire (`pipewire-devel`, `libspa-devel`), Wayland client libraries, D-Bus
- Build Tools: `cmake`, `pkg-config`, `just`, `git`

**Production:**
- Target Platform: Linux x86_64 on Wayland desktop environment (GNOME, KDE Plasma, Sway, Hyprland, etc.)
- GPU Drivers: Vulkan 1.1+ capable driver (RADV, ANV, NVIDIA) or OpenGL ES 2.0/3.0 driver
- Audio: PipeWire 0.3 daemon running on the host
- Sandboxing / Packaging: Native binary, Flatpak (`io.github.luohoa97.Cordial`), or Distrobox container

---

*Stack analysis: 2026-08-20*
