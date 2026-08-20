# Codebase Concerns

**Analysis Date:** 2026-08-20

## Tech Debt

**Bionic Linker Host Adaptation:**
- Issue: Upstream `mcpelauncher-linker` compiled with `-DPATH_MAX=256`, causing potential buffer overflows with `realpath` and long file paths.
- Files: `native/CMakeLists.txt`, `third_party/mcpelauncher-linker/`
- Impact: Potential crash with fortified libc calls inside containers or when path lengths exceed 256 bytes.
- Fix approach: Explicitly overridden to `-DPATH_MAX=4096` in `native/CMakeLists.txt`; ongoing maintenance required when updating submodule.

**WebKitGTK In-Process Host Dependency:**
- Issue: Compiling `crates/cordial-shell` with the `webview` feature requires `webkitgtk6.0-devel`, which is unavailable on immutable Linux distributions like Fedora Silverblue without layering packages.
- Files: `crates/cordial-shell/Cargo.toml`, `crates/cordial-runtime/Cargo.toml`
- Impact: Users on immutable hosts cannot build web authentication views natively without using Distrobox (`just build distrobox`).
- Fix approach: Gated behind optional `webview` feature flag; containerized builds provide WebKit headers while runtime dynamically links to host libraries.

**Audio Routing In-Experience (FMOD vs OpenSL ES):**
- Issue: While OpenSL ES bridge feeds PipeWire for system audio, Roblox logs mention `org.fmod.AudioDevice` (FMOD Java `AudioTrack` output path) in some experience contexts.
- Files: `native/opensles.cpp`, `native/audio_classes.cpp`, `crates/cordial-runtime/src/android/looper.rs`
- Impact: Potential loss of in-experience 3D audio if FMOD bypasses OpenSL ES.
- Fix approach: Continue evaluating JNI AudioTrack calls against logcat traces (`docs/traces/`) and implement Java AudioTrack bindings if required.

## Known Bugs

**13-Second Present Count Throttle:**
- Symptoms: `vkQueuePresentKHR` frame presentation drops to exactly 1.0 frame/sec after 13 seconds without user input.
- Files: `crates/cordial-runtime/src/android/wayland.rs`, `crates/cordial-runtime/src/android/input.rs`
- Trigger: Inactivity on mouse/keyboard inputs.
- Workaround: Continuous synthetic or user input maintains full 60.0 Hz vsync lock; do not treat raw present counts as a framerate metric.

**Wayland Pointer Lock Sensitivity Across Compositors:**
- Symptoms: Relative pointer motion may fail to turn the camera depending on compositor pointer constraint protocol support.
- Files: `crates/cordial-runtime/src/android/wayland.rs`, `crates/cordial-runtime/src/android/input.rs`
- Trigger: Pointer lock requested while cursor is outside the active surface.
- Workaround: Ensure cursor is centered over the rendering canvas before activating pointer lock.

## Security Considerations

**Untrusted Plugin Execution:**
- Risk: Malicious or buggy plugins attempting host privilege escalation or unauthorized file access.
- Files: `crates/cordial-plugins/src/sandbox.rs`, `crates/cordial-plugins/src/broker.rs`
- Current mitigation: Plugins execute under Deno with zero default permissions, wrapped inside Bubblewrap (`bwrap`) Linux sandboxes (ADR-018); capabilities are mediated by the host broker (ADR-007).
- Recommendations: Maintain strict manifest capability audits and deny raw file descriptor passing.

**Profile Isolation and Data Corruption:**
- Risk: Concurrent instances accessing the same profile directory corrupting SQLite cookie storage or client settings.
- Files: `crates/cordial-shell/src/profile.rs`, `crates/cordial-runtime/src/profile.rs`
- Current mitigation: Strict POSIX `flock` held on the profile directory for the process lifetime (ADR-012).
- Recommendations: Maintain advisory locking invariants across all launch tools and test runners.

## Performance Bottlenecks

**Verbose JNI Tracing (`CORDIAL_JNI_TRACE`):**
- Problem: Tracing logs every JNI method invocation, causing severe frame drops.
- Files: `native/CMakeLists.txt`, `native/jnivm_logger/log.cpp`
- Cause: Roblox engine polls `MotionEvent.getRawX` / `getRawY` continuously per frame; unbuffered stdout writes saturate I/O.
- Improvement path: Disabled by default; use only for targeted debugging and inventory discovery.

## Fragile Areas

**Bionic / Glibc Boundary ABI Alignment:**
- Files: `crates/cordial-runtime/src/bionic/`, `native/shim.cpp`, `crates/cordial-runtime/src/symtab.rs`
- Why fragile: Struct layouts, TLS descriptors, and signal handlers differ slightly between Android Bionic and Linux glibc.
- Safe modification: Check data symbol sizes with `readelf -sW` and cross-reference with `docs/analysis/undefined-symbols.tsv`.
- Test coverage: Validated through automated unit tests and runtime smoke tests.

## Scaling Limits

**Concurrent Instances:**
- Current capacity: Multiple independent instances on distinct profile directories.
- Limit: Limited by host GPU VRAM and CPU capacity.
- Scaling path: Single instance per profile enforced by design.

## Dependencies at Risk

**AOSP Bionic Linker Port (`third_party/mcpelauncher-linker`):**
- Risk: Upstream repository maintenance cadence is low; relies on custom C++11 patches.
- Impact: New Roblox engine builds requiring newer Android ELF features may require local linker adjustments.
- Migration plan: Maintain minimal local overrides in `native/CMakeLists.txt` and track upstream changes.

## Missing Critical Features

**Per-Profile Network Namespace Isolation:**
- Problem: Currently, VPN enforcement (`vpn-required` mode in ADR-016) operates as a pre-launch gate checking `pvpn status` rather than an isolated network namespace.
- Blocks: True network isolation between concurrent profiles running simultaneously on different VPN interfaces.

## Test Coverage Gaps

**In-Experience 3D Graphics & Camera Verification:**
- What's not tested: Automated CI verification of 3D scene rendering, shader compilation, and camera turning inside active game experiences.
- Files: `crates/cordial-runtime/src/android/vulkan.rs`, `crates/cordial-runtime/src/android/gl.rs`
- Risk: Regressions in graphical rendering or camera controls require manual user testing with a live Roblox account.
- Priority: Medium (requires manual validation with dedicated test accounts).

---

*Concerns audit: 2026-08-20*
