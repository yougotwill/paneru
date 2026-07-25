# External Integrations

**Analysis Date:** 2026-07-25

Paneru is a fully local, single-process macOS window manager. **There are no network services, no HTTP clients, no databases, no auth providers, and no webhooks.** Verified: no `reqwest`/`hyper`/`tokio`/`TcpStream` usage anywhere in `src/`. All external surface area is (a) macOS system frameworks via FFI and (b) a local Unix-socket IPC protocol.

## APIs & External Services

**macOS system frameworks (all local, in-process FFI):**

- **SkyLight (private framework)** — window-server/compositor internals; the core of window and Space management.
  - Linked via `#[link(name = "SkyLight", kind = "framework")]` in `src/manager/skylight.rs:11` and `src/platform/notify.rs:15`; private framework search path emitted by `build.rs`.
  - Window queries: `SLSCopyWindowsWithOptionsAndTags`, `SLSWindowQueryWindows` + iterator APIs (`SLSWindowIteratorAdvance/GetWindowID/GetParentID/GetTags/GetAttributes`) — used by `WindowManagerOS` in `src/manager.rs` and `src/manager/windows.rs`.
  - Spaces: `SLSManagedDisplayGetCurrentSpace`, `SLSSpaceGetType` (0=user, 4=fullscreen, 2=system), `SLSCopyManagedDisplaySpaces`, `SLSGetSpaceManagementMode` (checks "Displays have separate Spaces"), `SLSCopyActiveMenuBarDisplayIdentifier`, `SLSGetDisplayMenubarHeight`.
  - Process/focus control: `_SLPSGetFrontProcess`, `_SLPSSetFrontProcessWithOptions`, `SLSGetConnectionIDForPSN`, `SLPSPostEventRecordTo` (synthetic input to a target PSN).
  - Misc: `SLSMainConnectionID`, `SLSFindWindowAndOwner` (window under cursor), `SLSGetCurrentCursorLocation`, `SLSCopyAssociatedWindows`, `SLSSetWindowListBrightness` (focus dimming), `CGDisplayCreateUUIDFromDisplayID` / `CGDisplayGetDisplayIDFromUUID`.
  - Space-change notifications: `SLSRegisterConnectionNotifyProc` for `SpaceCreated`, `SpaceDestroyed`, `SpaceCurrentChanged`, `SpaceWindowDestroyed` (`src/platform/notify.rs:49`; event IDs informed by yabai/hammerspoon undocumented-spaces headers, see comments at `src/platform/notify.rs:155`).
  - Runtime symbol resolution: `SLSWindowIteratorGetCornerRadii` is loaded with `libc::dlsym` because it only exists on macOS 26 (`src/manager/windows.rs:715`) — pattern to copy for version-gated private APIs.

- **Accessibility API (HIServices / AXUIElement)** — per-window control and event observation; requires user-granted Accessibility permission.
  - Permission check/prompt: `AXIsProcessTrusted`, `AXIsProcessTrustedWithOptions` (`check_ax_privilege`/`request_ax_privilege` in `src/manager.rs`; gated at startup in `src/platform.rs:209` and `src/main.rs:155`).
  - Observers: `AXObserverCreate` / `AXObserverAddNotification` / `AXObserverRemoveNotification` declared in `src/platform.rs:83`; run-loop source plumbing in `src/util.rs` (`add_run_loop`/`remove_run_loop`).
  - Attribute/action FFI: `AXUIElementCopyAttributeValue`, `AXUIElementSetAttributeValue`, `AXUIElementPerformAction`, `_AXUIElementGetWindow`, `_AXUIElementCreateWithRemoteToken` (windows on inactive Spaces) in `src/manager/skylight.rs`.
  - Mission Control detection: `AXObserver` on the **Dock process** for `AXExposeShowAllWindows`, `AXExposeShowFrontWindows`, `AXExposeShowDesktop`, `AXExposeExit` (`src/platform/mission_control.rs:52`).

- **AppKit (`objc2-app-kit`)** — app lifecycle, notifications, overlays.
  - `NSApplication` accessory-mode app + manual Cocoa event pump (`PlatformCallbacks::pump_cocoa_event_loop`, `src/platform.rs:237`). Must run on the main thread (`MainThreadMarker`).
  - `NSWorkspace` + `NSDistributedNotificationCenter` observers via a Rust-defined Objective-C class (`define_class!` `WorkspaceObserver`, `src/platform/workspace.rs:25`): active space/display change, app hide/unhide, system wake.
  - UI surfaces: menu-bar extra (`src/menubar.rs`, `MenuBarManager`), border/flash overlays (`src/overlay.rs`, `OverlayManager` — `NSWindow`/`NSView`/`NSBezierPath`/`NSColor`/`CALayer`), accessibility-permission alert (`src/accessibility_prompt.rs`, `NSAlert`).

- **CoreGraphics (`objc2-core-graphics`)** — input and displays.
  - `CGEventTap` at `HIDEventTap` level intercepting mouse buttons/motion, scroll wheel, `NSEventType::Gesture` (trackpad swipes via `NSTouch`), and key-down (`src/platform/input.rs:96`); this is what implements keybindings, focus-follows-mouse, and swipe gestures.
  - Display reconfiguration callback `CGDisplayRegisterReconfigurationCallback` (`src/platform/display.rs:54`).
  - Display/window queries: `CGGetActiveDisplayList`, `CGDisplayBounds`, `CGWindowListCopyWindowInfo`, cursor control `CGWarpMouseCursorPosition` / `CGAssociateMouseAndMouseCursorPosition` (`src/manager.rs`).

- **Carbon framework (HIToolbox + Process Manager)** — legacy but still-required C APIs.
  - Process lifecycle events: `GetApplicationEventTarget` + `InstallEventHandler` for app launched/terminated/front-switched (`src/platform/process.rs:35`).
  - Keyboard layout translation: `TISCopyCurrentASCIICapableKeyboardLayoutInputSource`, `TISGetInputSourceProperty`, `UCKeyTranslate` — converts config key names to keycodes (`src/config.rs:1380`).
  - `ProcessSerialNumber` type re-exported from `src/platform/process.rs:21`.

- **Foundation / CoreFoundation (`objc2-foundation`, `objc2-core-foundation`)** — `CFRunLoop` modes for observers, `CFMachPort` event-tap port, `NSProcessInfo.operatingSystemVersion` cached by `macos_major_version()` (`src/platform.rs:300`), `_NSGetExecutablePath` for self-location (`src/util.rs:312`).

- **FSEvents** — config hot-reload via the `notify` crate (`macos_fsevent` feature); watcher created in `WindowManagerOS::setup_config_watcher` (`src/manager.rs`), held as a non-send Bevy resource (`src/ecs.rs:552`), delivers `Event::ConfigRefresh`.

- **launchd** — service management via the `launchctl` crate.
  - Label `com.github.karinushka.paneru` (`src/platform/service.rs:13`); user agent plist written to `~/Library/LaunchAgents/` from template `assets/launchd.plist`.
  - Managed through CLI subcommands `install|uninstall|start|stop|restart` (`src/main.rs:170`); in-daemon restart spawns a detached `paneru restart` (`Service::request_restart`, `src/platform/service.rs:180`).

## Data Storage

**Databases:**
- None. All live state is in the Bevy ECS `World` (in-memory).

**File Storage:**
- Local filesystem only:
  - Config TOML — `$PANERU_CONFIG`, `$HOME/.paneru`, `$HOME/.paneru.toml`, or `$XDG_CONFIG_HOME/paneru/paneru.toml` (`src/config.rs:95`).
  - Session/state snapshot — XDG state dir `paneru/` via `PaneruState::default_state_file_path()` (`src/ecs/state.rs:335`), loaded at startup (`src/ecs.rs:574`), used for session restore.
  - Logs — `/tmp/paneru.log`, `/tmp/paneru.err.log` (launchd-managed) and stderr tracing.

**Caching:**
- None beyond in-process caches (`OnceLock` for macOS version, `LazyLock` for config path).

## Authentication & Identity

**Auth Provider:**
- None. The only security gate is the macOS **Accessibility permission** (TCC), checked with `AXIsProcessTrusted` and requested via `AXIsProcessTrustedWithOptions` (`src/manager.rs`); the app waits in a menu-bar-only mode until granted (`src/main.rs:186`, `wait_for_accessibility`).

## Monitoring & Observability

**Error Tracking:**
- None (no Sentry or similar).

**Logs:**
- `tracing` + `tracing-subscriber` to stderr, compact format with file/line (`src/main.rs:126`); filtered by `RUST_LOG` (default `info`). When running under launchd, stderr/stdout land in `/tmp/paneru.err.log` / `/tmp/paneru.log`.

## CI/CD & Deployment

**Hosting:**
- Distributed as a standalone binary (GitHub Releases) and as a Nix flake package (`nix/package.nix`).

**CI Pipeline:**
- GitHub Actions: `.github/workflows/rust.yml` — on push/PR to `main`, `macos-latest` runner, `cargo build --release`, artifact upload. No external services.

**System integration modules:**
- nix-darwin module `services.paneru` (`nix/darwin.nix`) and home-manager module (`nix/home.nix`) — install the binary, write the launchd agent, and optionally generate `paneru.toml`; module behavior verified by `nix/checks.nix`.

## Environment Configuration

**Required env vars:**
- None strictly required. Optional: `PANERU_CONFIG` (config path), `XDG_CONFIG_HOME` / `HOME` (config/state discovery), `RUST_LOG` (log filter), `NO_COLOR` (set by launchd plist).

**Secrets location:**
- Not applicable — the project uses no secrets, tokens, or credentials. No `.env` files exist.

## IPC & Local Interfaces (in place of Webhooks)

**Incoming — Unix domain socket `/tmp/paneru.socket`** (`src/reader.rs`):

Wire protocol: 4-byte little-endian `u32` length, then NUL-separated argv strings (`CommandReader::send_socket_request`, `src/reader.rs:59`). Three request classes, dispatched into the Bevy world as events:

1. **Commands** — `paneru send-cmd <args>` → parsed by `config::parse_command` into `commands::Command` (`src/commands.rs`), sent as `Event::Command`. E.g. `paneru send-cmd window virtualnum 3`.
2. **Queries** — `paneru query state|virtual-workspaces|active --json` → `Event::StateQuery` with an mpsc reply channel; daemon responds within a 2-second timeout with a JSON document + newline (`src/reader.rs:130`, `src/ecs/state.rs`). Document format specified in `QUERY_AND_SUBSCRIBE_FORMAT.md` (versioned: `"version": 1`).
3. **Subscriptions** — `paneru subscribe --json` → long-lived nonblocking socket registered via `Event::StateSubscribe`; daemon streams line-delimited JSON events: `virtual_workspace_changed`, `windows_changed`, `window_focused`, `window_title_changed`, `display_changed`. Intended for status-bar integrations such as SketchyBar (`QUERY_AND_SUBSCRIBE_FORMAT.md:149`). Events are coalesced per ECS tick and deduplicated.

**Incoming — OS event streams:** CGEventTap, AXObservers, NSWorkspace/distributed notifications, Carbon process events, `SLSRegisterConnectionNotifyProc`, `CGDisplayReconfigurationCallback`, FSEvents — all normalized into the internal `Event` enum (`src/events.rs:20`) and fed to the ECS loop through `EventSender`.

**Outgoing:**
- Responses/streams on the Unix socket only. No outbound network calls, no webhooks, no callbacks to external processes (synthetic input via `SLPSPostEventRecordTo` goes to other local apps' event queues, not over IPC).

---

*Integration audit: 2026-07-25*
