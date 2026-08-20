# Testing Patterns

**Analysis Date:** 2026-08-20

## Test Framework

**Runner:**
- Rust: Built-in `cargo test` harness
- C++: Custom post-build test runner in CMake (`native/pipewire_backend_test.cpp`)

**Assertion Library:**
- Rust standard library macros (`assert!`, `assert_eq!`, `assert_ne!`)
- C++ standard assertions (`assert`, custom test macros in `pipewire_backend_test.cpp`)

**Run Commands:**
```bash
cargo test --workspace                # Run all unit and integration tests across the workspace
cargo test -p cordial-update          # Run tests for specific crate
cargo test -- --nocapture             # Run tests with console output
just test-audio                       # Run native PipeWire buffer tests
```

## Test File Organization

**Location:**
- Unit tests: Co-located inside source files under `#[cfg(test)] mod tests`
- Native tests: Co-located in `native/` alongside source files (e.g. `native/pipewire_backend_test.cpp`)

**Naming:**
- Inline test modules: `mod tests`
- Test functions: `test_*` or descriptive snake_case names (e.g. `test_version_parsing`, `test_sha256_verification`)

**Structure:**
```
crates/cordial-update/src/
├── version.rs             # Source logic + inline #[cfg(test)] mod tests
├── download.rs            # Source logic + inline #[cfg(test)] mod tests
└── settings.rs            # Source logic + inline #[cfg(test)] mod tests
```

## Test Structure

**Suite Organization:**
```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_semantic_version_correctly() {
        let version_str = "0.6.0";
        let parsed = parse_version(version_str).expect("valid semver");
        assert_eq!(parsed.major, 0);
        assert_eq!(parsed.minor, 6);
    }
}
```

**Patterns:**
- Setup: Creating temporary directories or mock profile roots via `tempfile` / environment overrides (`CORDIAL_PROFILE_ROOT`).
- Execution: Invoking pure parsing or state transformation functions.
- Assertion: Checking expected return codes, parsed values, and file system states.

## Mocking

**Framework:**
- Manual test doubles and synthetic environment variable injection.

**Patterns:**
- Environment redirection: Setting `XDG_DATA_HOME` or `CORDIAL_PROFILE_ROOT` to temporary test paths.
- Standalone audio test: Simulating PipeWire buffer fill and underrun states without a live audio daemon (`native/pipewire_backend_test.cpp`).

**What to Mock:**
- External web endpoints (simulate CDN JSON responses with test fixtures).
- File system paths for profile storage during isolated test runs.
- System D-Bus services when verifying fallback paths.

**What NOT to Mock:**
- Cryptographic hash verification (`sha2`, `minisign`).
- Actual ELF loading and symbol table resolution logic.
- Native libc/bionic bridging layers.

## Fixtures and Factories

**Test Data:**
- Test JSON snippets for FastFlags, client settings, and plugin manifests embedded directly in test modules.
- Sample version strings and APK header slices.

**Location:**
- Embedded inline in `mod tests` blocks or loaded from `docs/analysis/` and test fixtures.

## Coverage

**Requirements:**
- High coverage on security-sensitive components: cryptographic signing, package unpacking, version comparison, and profile lock arbitration.

**View Coverage:**
```bash
cargo test --workspace
```

## Test Types

**Unit Tests:**
- Fast, deterministic tests verifying business logic, JSON parsing, and cryptographic checks (`cordial-update`, `cordial-plugins`, `cordial-shell`).

**Integration Tests:**
- Verifies inter-crate behavior, profile locking (`flock`), and virtual symbol routing table generation.

**Live Execution & Verification Harnesses:**
- `tools/join-run.sh` / `just client`: Launches full client against a test profile with an APK supplied by the user.

## Common Patterns

**Error Testing:**
```rust
#[test]
fn rejects_invalid_signature() {
    let bad_data = b"tampered payload";
    let result = verify_signature(bad_data, &TEST_KEY);
    assert!(result.is_err());
}
```

**Environment Isolation:**
```rust
#[test]
fn profile_lock_prevents_duplicate_instance() {
    let tmp = tempfile::tempdir().unwrap();
    let profile1 = Profile::open(tmp.path()).unwrap();
    let profile2 = Profile::open(tmp.path());
    assert!(profile2.is_err());
}
```

---

*Testing analysis: 2026-08-20*
