# External Integrations

**Analysis Date:** 2026-08-20

## APIs & External Services

**Roblox Client Settings (FastFlags):**
- Service: Roblox Client Settings CDN (`clientsettings.api.roblox.com` / `clientsettingscdn.roblox.com`)
  - Purpose: Fetches official engine configuration flags, dynamic feature toggles, and client settings.
  - SDK/Client: `ureq` in `crates/cordial-runtime/src/client_settings.rs`
  - Auth: None (public CDN endpoint), user flag overrides are merged locally in memory

**Roblox APK Distribution:**
- Service: Roblox Android APK deployment endpoints
  - Purpose: Downloads official Android x86-64 base APK packages on user demand (ADR-015; Cordial ships no proprietary binaries).
  - SDK/Client: `ureq` in `crates/cordial-update/src/download.rs`
  - Auth: None (public APK endpoints)

**Roblox Web Authentication & WebAuthn:**
- Service: Roblox Login & Account Portal
  - Purpose: Authentication flow, passkey/WebAuthn handling, Robux transactions, and account management sheets.
  - SDK/Client: WebKitGTK 6.0 in `crates/cordial-shell/src/webview.rs` and cookie bridge in `crates/cordial-runtime/src/cookies.rs`
  - Auth: `.ROBLOSECURITY` session cookies extracted from WebKit cookie jar

**Discord Rich Presence:**
- Service: Discord IPC (`/run/user/$UID/discord-ipc-0`)
  - Purpose: Displays game activity, experience name, elapsed time, and icon in Discord.
  - SDK/Client: `crates/cordial-plugins/src/presence.rs` and `plugins/discord-presence`
  - Auth: Local UNIX domain socket handshaking

**GitHub Releases:**
- Service: GitHub Releases API (`api.github.com/repos/luohoa97/cordial/releases`)
  - Purpose: Fetches update metadata, release notes, and version tags.
  - SDK/Client: `ureq` in `crates/cordial-update/src/changelog.rs`
  - Auth: None (public read-only)

## Data Storage

**Databases:**
- None (No embedded SQL database used). State is managed via structured JSON documents.

**File Storage:**
- Local Filesystem Only:
  - Per-profile directory storage under `$XDG_DATA_HOME/cordial/profiles/<profile_name>/` (ADR-012, ADR-013)
  - Engine library cache under `$XDG_CACHE_HOME/cordial/lib/x86_64/` (ADR-015)
  - Profile state lock descriptor held via `flock` (`crates/cordial-shell/src/profile.rs`, `crates/cordial-runtime/src/profile.rs`)

**Caching:**
- Local APK & Shared Library Cache:
  - Extracted `libroblox.so` cached and verified against SHA-256 hash and version stamps (`crates/cordial-update/src/cache.rs`)

## Authentication & Identity

**Auth Provider:**
- Custom WebKitGTK Embedded Web Authentication Bridge:
  - Approach: Renders Roblox web login in an `AdwDialog` sheet (`crates/cordial-shell/src/webview.rs`).
  - Cookie Interception: Reads `.ROBLOSECURITY` cookie from `WebKitCookieManager` and synchronizes it with the engine's Android cookie store (`native/cookies.cpp`, `crates/cordial-runtime/src/cookies.rs`).
  - Passkey / WebAuthn: Backed by WebKitGTK system credentials.

**Secret Storage:**
- Freedesktop Secret Service API (`org.freedesktop.secrets` over D-Bus via `zbus` in `crates/cordial-runtime/src/secrets.rs`):
  - Stores encrypted login tokens (`.ROBLOSECURITY`) securely in GNOME Keyring / KWallet.

**Device Identity Emulation:**
- Android Device Metadata (`crates/cordial-runtime/src/identity.rs`, `native/init_params.cpp`):
  - Emulates consistent hardware identifiers, manufacturer strings, OS build numbers, and install GUIDs required by Roblox telemetry.

## Monitoring & Observability

**Error Tracking:**
- None (No remote telemetry or third-party error tracking SDKs; strictly privacy-preserving).

**Logs & Diagnostics:**
- Android `liblog` / Logcat Interceptor:
  - Captures engine log messages formatted via `__android_log_print` / `__android_log_write` (`native/liblog.cpp`).
  - Routes logs to standard output with colored prefix tagging (`[DFLog::...]`, `[Roblox]`).
- JNI Unresolved Call Tracker:
  - Intercepts unresolved Java classes and methods via `libjnivm` logger (`native/jnivm_logger/log.cpp`).
  - Exports class dump via `--dump-classes <file>` and generates unresolved symbol reports (`crates/cordial-runtime/src/unimplemented.rs`).

## CI/CD & Deployment

**Hosting:**
- GitHub repository (`luohoa97/cordial`)
- Flathub / Flatpak distribution repository

**CI Pipeline:**
- GitHub Actions:
  - `.github/workflows/flatpak.yml` - Builds and validates Flatpak package against `org.gnome.Platform//47` SDK.
- Containerized Development:
  - Distrobox / Toolbox environment (`just build distrobox`) ensuring consistent build toolchains with host package parity.

## Environment Configuration

**Required env vars:**
- `WAYLAND_DISPLAY` - Wayland compositor socket name (required for window surface presentation)
- `XDG_DATA_HOME` - Base directory for user profile storage (defaults to `~/.local/share`)
- `XDG_CONFIG_HOME` - Base directory for shell configuration (defaults to `~/.config`)
- `XDG_CACHE_HOME` - Base directory for extracted engine binaries (defaults to `~/.cache`)

**Optional diagnostics & switches:**
- `CORDIAL_PROFILE_ROOT` - Relocates profile directories for testing
- `CORDIAL_TRACE_TEXT=1` - Dumps Wayland text input events (IME, keypresses)
- `CORDIAL_TRACE_PATHS=1` - Traces file path queries across bionic/glibc shims
- `CORDIAL_JNI_TRACE=1` - Verbose trace of all JNI operations (build-time / run-time)

**Secrets location:**
- System Keyring via Freedesktop Secret Service (`org.freedesktop.secrets`), with fallback to encrypted local profile storage.

## Webhooks & Callbacks

**Incoming:**
- Deep Links / Protocol Handler:
  - URI Schemes: `roblox://` and `roblox-player://` handled by `crates/cordial-shell/src/deep_link.rs` and dispatched to `crates/cordial-runtime/src/deeplink.rs`.
- D-Bus System Signals:
  - `org.freedesktop.NetworkManager` (`PropertiesChanged` signal on `/org/freedesktop/NetworkManager` for metered connection detection).
  - `org.a11y.atspi` accessibility tree queries.

**Outgoing:**
- D-Bus Method Calls:
  - `com.feralinteractive.GameMode` (`RegisterStatus` / `UnregisterStatus` for GameMode activation).
  - `org.freedesktop.secrets` (Session secret retrieval and storage).

---

*Integration audit: 2026-08-20*
