# Technology Stack

**Analysis Date:** 2026-07-25

## Languages

**Primary:**
- Rust (edition 2024, MSRV `1.89.0`) — entire codebase under `src/`. Declared in `Cargo.toml` (`edition = "2024"`, `rust-version = "1.89.0"`).

**Secondary:**
- Nix — build, packaging, and system modules: `flake.nix`, `nix/package.nix`, `nix/darwin.nix`, `nix/home.nix`, `nix/checks.nix`.
- XML (plist) — app bundle metadata `assets/Info.plist` (embedded into the binary) and launchd template `assets/launchd.plist`.
- Objective-C interop is done in Rust via `objc2::define_class!` (no .m files), e.g. `WorkspaceObserver` in `src/platform/workspace.rs:25`.

## Runtime

**Environment:**
- macOS only (`categories = ["os::macos-apis"]` in `Cargo.toml`; flake `systems = lib.platforms.darwin`). Single native binary `paneru` — no separate runtime, no VM, no async runtime (no tokio). Concurrency is Bevy's multi-threaded ECS scheduler plus a handful of `std::thread`s (`src/reader.rs:89`).
- Requires macOS SDK from Xcode Command Line Tools at build time (`build.rs` scans `/Library/Developer/CommandLineTools/SDKs`).
- Runtime requirements: Accessibility permission (`check_ax_privilege` in `src/manager.rs`, enforced in `src/platform.rs:209`) and "Displays have separate Spaces" enabled (`check_separate_spaces`, `src/platform.rs:215`).

**Package Manager:**
- Cargo — lockfile present and committed: `Cargo.lock`.
- Nix flakes as an alternative build/distribution channel — `flake.lock` present; package built with `pkgs.rustPlatform.buildRustPackage` in `nix/package.nix`.

## Frameworks

**Core:**
- Bevy `0.18.1` (locked in `Cargo.lock`) with `default-features = false`, features `["multi_threaded"]` — used purely as an ECS kernel with `MinimalPlugins`; no rendering, no winit, no Bevy window. App assembly in `setup_bevy_app` at `src/ecs.rs:535`. Custom plugins: `MouseEventsPlugin`, `ScrollEventsPlugin`, `WorkspaceEventsPlugin`, `LayoutEventsPlugin`, `FocusEventsPlugin`, `DisplayEventsPlugin` (`src/ecs.rs:553`).
- clap `4.6` (`cargo`, `derive` features) — CLI definition in `src/main.rs:42` (`Paneru` struct, `SubCmd` enum: `launch`, `install`, `uninstall`, `start`, `stop`, `restart`, `send-cmd`, `query`, `subscribe`).

**macOS FFI (objc2 ecosystem, all `0.3`/`0.6` lines):**
- `objc2` `0.6` (locked `0.6.4`), features `["catch-all", "exception"]` — Objective-C runtime interop.
- `objc2-app-kit` `0.3` — AppKit (features: `NSAlert`, `NSBezierPath`, `NSColor`, `NSGraphics`, `NSGraphicsContext`, `NSScreen`, `NSView`, `NSWindow`, `NSFont`, `NSAttributedString`, `NSParagraphStyle`, `NSStringDrawing`).
- `objc2-core-foundation` `0.3` (feature `CFRunLoop`), `objc2-core-graphics` `0.3`, `objc2-quartz-core` `0.3` (feature `CALayer`), `objc2-foundation` `0.3` (features `NSNotification`, `NSProcessInfo`, `NSAttributedString`, `NSDictionary`, `NSBundle`).
- `accessibility-sys` `0.2` — raw C bindings for the Accessibility (AXUIElement/AXObserver) API.
- Raw `extern "C"` blocks with `#[link(name = ..., kind = "framework")]` for frameworks without crate bindings: SkyLight (`src/manager/skylight.rs:11`, `src/platform/notify.rs:15`), Carbon (`src/config.rs:1380`, `src/platform/process.rs:35`), Foundation (`src/util.rs:312`).

**Testing:**
- Built-in `cargo test`; `mockall` `0.14` for trait mocking (`#[automock]` on `WindowManagerApi` in `src/manager.rs:81`; mocks exported as `MockProcessApi`/`MockWindowApi` for tests). Note: `mockall` is a regular dependency, not dev-only.
- Nix module test harness in `nix/checks.nix` (builds a nix-darwin configuration and lints the generated launchd plist with `plutil`/`jq`).

**Build/Dev:**
- `build.rs` — emits `cargo:rustc-link-search=framework=...` for every installed `MacOSX*.sdk`: `System/Library/PrivateFrameworks` (SkyLight) and `Carbon.framework/Versions/A/Frameworks` (HIToolbox). `nix/package.nix:33` patches this path to `pkgs.apple-sdk` for hermetic builds.
- `rust-toolchain.toml` — `channel = "stable"`, `profile = "default"`.
- `embed_plist` `1.2` — compile-time embeds `assets/Info.plist` into the binary (`src/main.rs:25`).
- Release profile: `codegen-units = 1`, `lto = true` (`Cargo.toml:17`).

## Key Dependencies

**Critical:**
- `bevy 0.18.1` — ECS backbone; all window/workspace state lives in the Bevy `World`.
- `objc2*` + `accessibility-sys` — the entire macOS bridge; see `src/manager/skylight.rs`, `src/platform/`.
- `tracing 0.1` + `tracing-subscriber 0.3` (`env-filter`) — logging; initialized in `src/main.rs:126` with `RUST_LOG` env filter defaulting to `info`, compact format to stderr.
- `notify 8.2` (feature `macos_fsevent`) — FSEvents-based config file hot-reload (`setup_config_watcher` in `src/manager.rs`, used at `src/ecs.rs:537`).
- `serde 1.0` + `serde_json 1.0` — JSON state documents for the query/subscribe IPC (`src/ecs/state.rs`, format in `QUERY_AND_SUBSCRIBE_FORMAT.md`).
- `toml 1.1` — user configuration parsing (`src/config.rs`).
- `launchctl 0.3` — launchd agent management (`src/platform/service.rs`).
- `clap 4.6` — CLI parsing for both the daemon and the socket client subcommands.

**Infrastructure:**
- `arc-swap 1.9` — lock-free sharing of passthrough keybindings with the CGEventTap callback (`src/platform/input.rs:31`).
- `bitflags 2.13` — `Modifiers` bitflags (`src/platform.rs:49`).
- `chrono 0.4` — timestamps (state document `timestamp` field).
- `ctrlc 3.5` (feature `termination`) — SIGINT/SIGTERM → `Event::Exit` (`src/main.rs:150`).
- `derive_more 2.1` (`deref`, `deref_mut`), `libc 0.2` (incl. `dlsym` for macOS-26-only symbols, `src/manager/windows.rs:715`), `num_enum 0.7` (`KnownCGSEvent` mapping in `src/platform/notify.rs`), `regex 1.12` (window-title matching in config), `scopeguard 1.2` (pinned handler cleanup guards), `stdext 0.3` (`function_name!`), `xdg 3.0` (config/state discovery).

## Configuration

**Environment:**
- `PANERU_CONFIG` — explicit path to config TOML (highest priority, `src/config.rs:96`).
- `XDG_CONFIG_HOME` / `HOME` — config discovery: `$HOME/.paneru`, `$HOME/.paneru.toml`, then XDG `paneru/paneru.toml` (`discover_configuration_file`, `src/config.rs:95`). A default config is auto-created if none exists (`src/config.rs:42`).
- `RUST_LOG` — tracing filter (`src/main.rs:127`).
- `NO_COLOR=1` — set by the launchd plist (`assets/launchd.plist:22`).
- No `.env` files; no secrets are used by this project.

**Build:**
- `Cargo.toml` (`[lints.clippy] pedantic = "warn"` — enforced project-wide; fix warnings with `cargo clippy`), `rust-toolchain.toml`, `build.rs`. No standalone `clippy.toml` or `rustfmt.toml` — stock `cargo fmt` settings.

## Platform Requirements

**Development:**
- macOS with Xcode Command Line Tools (SDK at `/Library/Developer/CommandLineTools/SDKs` for `build.rs`).
- Rust stable (pinned by `rust-toolchain.toml`; MSRV 1.89).
- Or `nix develop` — devShell provides `rustc`, `cargo`, `rustfmt`, `clippy` plus the built `paneru` (`flake.nix:34`).
- Common commands: `cargo build`, `cargo test`, `cargo clippy`, `cargo fmt`, `nix fmt .` (nixfmt-tree), `nix flake check` (runs `nix/checks.nix`).

**Production:**
- macOS (Apple Silicon primary target; flake exposes `aarch64-darwin` package as the default overlay output, `flake.nix:23`).
- Runs as a launchd user agent: label `com.github.karinushka.paneru`, plist at `~/Library/LaunchAgents/com.github.karinushka.paneru.plist`, logs at `/tmp/paneru.log` and `/tmp/paneru.err.log`, `Nice = -20`, `ProcessType = Interactive`, `KeepAlive.Crashed = true` (`src/platform/service.rs:197`, `assets/launchd.plist`).
- Distribution channels: cargo build, GitHub release binary, Nix flake package + `nix-darwin` module (`nix/darwin.nix`) + home-manager module (`nix/home.nix`).
- CI: GitHub Actions `.github/workflows/rust.yml` — `macos-latest` runner, `cargo build --release`, uploads `target/release/paneru` artifact. No test/clippy/fmt gates in CI currently.

---

*Stack analysis: 2026-07-25*
