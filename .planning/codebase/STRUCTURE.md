# Codebase Structure

**Analysis Date:** 2026-07-25

## Directory Layout

```text
mighty-perch/
├── src/                    # All Rust source (single binary crate `paneru`)
│   ├── main.rs             # Entry point: clap CLI, daemon bootstrap, accessibility gate
│   ├── ecs.rs              # Bevy app assembly, system/trigger registration, marker components
│   ├── ecs/                # ECS domain modules (layout, workspace, focus, …)
│   ├── manager.rs          # WindowManagerApi trait, WindowManagerOS, SkyLight window queries
│   ├── manager/            # OS bridge: app / windows / process / display / skylight FFI
│   ├── platform.rs         # PlatformCallbacks aggregate, Modifiers bitflags, type aliases
│   ├── platform/           # macOS event sources + launchd service
│   ├── commands.rs         # Command/Operation enums + PreUpdate command handler systems
│   ├── commands/           # State query (JSON) IPC handlers
│   ├── config.rs           # TOML config: discovery, parsing, hot reload, bindings, window rules
│   ├── config/             # Config sub-domains (decorations, padding, swipe)
│   ├── events.rs           # `Event` enum + `EventSender` mpsc wrapper
│   ├── errors.rs           # `Error` enum + crate-wide `Result<T>`
│   ├── reader.rs           # CommandReader: Unix socket `/tmp/paneru.socket` IPC
│   ├── menubar.rs          # NSStatusItem menu bar UI
│   ├── overlay.rs          # NSWindow overlays: focus border, dimming, flash messages
│   ├── accessibility_prompt.rs  # Accessibility permission setup dialog
│   ├── util.rs             # AXUIWrapper, MacResult, CF helpers, dark-mode check
│   ├── tests.rs            # Test module root: shared constants
│   └── tests/              # Integration-style ECS tests (harness, mocks, features)
├── assets/
│   ├── Info.plist          # Embedded into the binary via `embed_plist!` (src/main.rs:25)
│   └── launchd.plist       # Template for the launchd agent
├── nix/                    # Nix packaging: package.nix, darwin.nix, home.nix, checks.nix
├── .github/workflows/      # CI
├── images/                 # README images
├── Cargo.toml              # Crate manifest (bin `paneru`, clippy pedantic)
├── Cargo.lock              # Lockfile (committed)
├── build.rs                # Adds macOS SDK PrivateFrameworks/Carbon framework search paths
├── rust-toolchain.toml     # Pins `stable` channel, default profile
├── flake.nix / flake.lock  # Nix flake dev shell + package outputs
├── AGENTS.md               # AI-agent contribution guide (ECS patterns, main-thread rules)
├── ARCHITECTURE.md         # Human-written architecture overview (contributor docs)
├── CONFIGURATION.md        # Full configuration reference
├── QUERY_AND_SUBSCRIBE_FORMAT.md  # JSON schema for `query`/`subscribe` IPC output
├── README.md               # User-facing readme
└── LICENSE.txt             # MIT
```

## Directory Purposes

**`src/`:**
- Purpose: Entire application — single binary crate, no lib target.
- Contains: Flat parent modules (`foo.rs`) each paired with a same-named directory of submodules (`foo/`).
- Key files: `src/main.rs` (entry), `src/ecs.rs` (Bevy wiring), `src/manager.rs` (OS bridge), `src/platform.rs` (event sources).

**`src/ecs/` — ECS domain modules:**
- Purpose: All Bevy-side state and behavior.
- Contains:
  - `layout.rs` (2118 lines) — `LayoutStrip`, `Column`, `StackItem`, tiling math, `LayoutEventsPlugin`
  - `workspace.rs` — virtual workspace strips per macOS space, `WorkspaceEventsPlugin`
  - `systems.rs` — core lifecycle/animation/commit/event-pump systems
  - `triggers.rs` — message-reading trigger systems + observers
  - `params.rs` — `SystemParam` bundles (`Windows`, `ActiveDisplay`, `ActiveDisplayMut`, `GlobalState`)
  - `focus.rs` — `FocusHistory`, tier memory, `FocusEventsPlugin`
  - `mouse.rs` / `scroll.rs` — drag & swipe gesture systems + plugins
  - `display.rs` — display reconciliation, `FloatingLayer`, dock position, `DisplayEventsPlugin`
  - `state.rs` — `PaneruState` snapshot save/load, state queries
  - `restore.rs` — startup session restore (`SessionRestore` grace period)

**`src/manager/` — OS bridge implementations:**
- Purpose: Mockable traits + macOS implementations of window/app/process/display access.
- Contains: `app.rs` (`ApplicationOS`, AX observers), `windows.rs` (`WindowOS`, AX position/size), `process.rs` (`ProcessOS`), `display.rs` (`Display`), `skylight.rs` (private `SLS*`/`_AX*` extern fns).
- Rule: ECS code imports traits from `src/manager.rs` re-exports; only `*OS` types touch FFI.

**`src/platform/` — macOS event sources:**
- Purpose: One file per OS event source, all owned by `PlatformCallbacks` (`src/platform.rs:140`).
- Contains: `input.rs` (CGEventTap keyboard/mouse/gestures), `process.rs` (Carbon PSN lifecycle), `workspace.rs` (`NSWorkspace` observers), `display.rs` (CoreGraphics display reconfiguration), `mission_control.rs` (AXObserver on the Dock), `notify.rs` (distributed notifications: dock/theme/wake), `service.rs` (launchd agent management).

**`src/commands/` + `src/commands.rs`:**
- Purpose: User command surface — `Command`/`Operation`/`Direction` enums and their handler systems (registered in `PreUpdate` by `register_commands`, `src/commands.rs:137`).
- Contains: `query.rs` — JSON responses for `paneru query …` / `paneru subscribe`.

**`src/config/` + `src/config.rs`:**
- Purpose: TOML configuration — file discovery (`PANERU_CONFIG`, `~/.paneru`, `~/.paneru.toml`, XDG), default-file creation, hot reload, keybindings parsing (`parse_command` doubles as the socket command parser), `[windows]` rules.
- Contains: `decorations.rs` (borders/radius/dimming), `padding.rs`, `swipe.rs` (gesture direction config).

**`src/tests/` — ECS integration tests:**
- Purpose: Drive a real Bevy `World` against mocked OS traits.
- Contains: `harness.rs` (`TestHarness` — builds the app with all plugins/systems minus NonSend OS resources), `mocks.rs` (`MockState` factory for mock WM/process/app/window), feature suites `interaction.rs`, `tiling.rs`, `tabs.rs`, `display.rs`, `state.rs`, `session_restore.rs`.
- Registered via `src/tests.rs` (`#[cfg(test)] mod tests;` in `src/main.rs:23`).

**`assets/`:**
- Purpose: macOS bundle metadata compiled into the binary.
- Generated: No (hand-maintained).
- Committed: Yes. `Info.plist` is embedded at compile time by `embed_plist::embed_info_plist!` (`src/main.rs:25`).

**`nix/`:**
- Purpose: Nix build/dev tooling (`package.nix`, `darwin.nix` nix-darwin module, `home.nix` home-manager module, `checks.nix`).
- Committed: Yes; see `nix/README.md`.

**`.planning/`:**
- Purpose: GSD planning artifacts (including this codebase map). Not part of the application.

## Key File Locations

**Entry Points:**
- `src/main.rs`: `main()` (line 125) — CLI dispatch; `SubCmd` enum (line 58)
- `src/ecs.rs`: `setup_bevy_app` (line 535), `register_systems` (line 59), `register_triggers` (line 163)
- `src/reader.rs`: `CommandReader::runner` (line 104) — daemon-side socket loop

**Configuration:**
- `Cargo.toml`: dependencies (Bevy 0.18 `MinimalPlugins`-style, `default-features = false`, `multi_threaded`), `[lints.clippy] pedantic = "warn"`, release profile (`lto`, `codegen-units = 1`)
- `build.rs`: framework search paths for PrivateFrameworks/Carbon under `/Library/Developer/CommandLineTools/SDKs`
- `rust-toolchain.toml`: `stable` toolchain
- `src/config.rs`: `CONFIGURATION_FILE` lazy path (line 42)
- `assets/Info.plist`, `assets/launchd.plist`

**Core Logic:**
- `src/ecs/layout.rs`: tiling (`LayoutStrip` struct at line 239)
- `src/ecs/workspace.rs`: virtual workspaces (`WorkspaceEventsPlugin` at line 34)
- `src/ecs/systems.rs`: frame loop & commit (`pump_events` line 584, `commit_window_position` line 932, `commit_window_size` line 968)
- `src/ecs/triggers.rs`: reactive handlers (`front_switched_trigger` line 55, `window_focused_trigger` line 178)
- `src/manager.rs`: `WindowManagerApi` (line 81), `WindowManagerOS` (line 196)
- `src/manager/skylight.rs`: private SkyLight FFI declarations

**Testing:**
- `src/tests/harness.rs`: `TestHarness` (line 30)
- `src/tests/mocks.rs`: `MockState` and mock builders
- `src/tests.rs`: shared `TEST_*` constants

**Docs (root):**
- `ARCHITECTURE.md` — contributor architecture doc (mermaid data-flow)
- `CONFIGURATION.md` — config reference
- `QUERY_AND_SUBSCRIBE_FORMAT.md` — IPC JSON schema
- `AGENTS.md` — agent/AI contributor rules

## Naming Conventions

**Files:**
- Parent-module pattern (Rust 2018 style): `foo.rs` + `foo/` directory — e.g. `src/ecs.rs` + `src/ecs/`, `src/manager.rs` + `src/manager/`. There are no `mod.rs` files.
- Test suites: feature name in `src/tests/` — `tiling.rs`, `tabs.rs`, `session_restore.rs`.
- One OS event source per file in `src/platform/`: `input.rs`, `notify.rs`, …

**Types:**
- OS API traits: `*Api` — `WindowManagerApi`, `WindowApi`, `ApplicationApi`, `ProcessApi` (all `#[automock]`).
- macOS implementations: `*OS` — `WindowManagerOS`, `WindowOS`, `ApplicationOS`, `ProcessOS`.
- Marker components: `*Marker` — `FocusedMarker`, `ActiveWorkspaceMarker`, `ReshuffleAroundMarker`, `NativeFullscreenMarker`.
- Bevy plugins: `*EventsPlugin` — `LayoutEventsPlugin`, `WorkspaceEventsPlugin`, `FocusEventsPlugin`, `MouseEventsPlugin`, `ScrollEventsPlugin`, `DisplayEventsPlugin`.
- Bevy event structs for `commands.trigger(..)`: `*Trigger` — `SpawnWindowTrigger`, `SendMessageTrigger` (`src/ecs.rs:437-443`).
- SystemParam bundles: plain nouns — `Windows`, `ActiveDisplay`, `ActiveDisplayMut`, `GlobalState` (`src/ecs/params.rs`).
- Coordinate types: `Origin`/`Size` (both `IVec2` aliases, `src/manager.rs:59-60`), with `*_from`/`*_to` converters (`origin_from`, `size_from`, `irect_from`).

**Functions:**
- Systems: verb phrases — `gather_displays`, `commit_window_position`, `reshuffle_layout_strip`.
- Trigger systems/observers: `*_trigger` — `window_focused_trigger`, `spawn_window_trigger`.
- Event handlers in command layer: `*_handler` — `command_quit_handler`, `workspace_change_handler`.
- FFI wrappers: keep the C symbol name (`SLSMainConnectionID`, `_AXUIElementCreateWithRemoteToken`) in `src/manager/skylight.rs`.

**Variables/fields:** `snake_case` throughout; acronyms kept short (`main_cid`, `psn`, `mtm` for `MainThreadMarker`).

## Where to Add New Code

**New ECS system (core lifecycle/frame behavior):**
- Primary code: `src/ecs/systems.rs`
- Registration: `register_systems` in `src/ecs.rs:59` — pick the schedule (`PreUpdate` for event ingestion, `Update` for logic, `PostUpdate` for OS write-back), add `.chain()` ordering and run conditions as needed.
- Tests: `src/tests/<feature>.rs` via `TestHarness`.

**New reactive handler for an OS event:**
- Add an `Event` variant in `src/events.rs:20`.
- Emit it from the relevant `src/platform/*.rs` handler (wire new sources into `PlatformCallbacks::setup_handlers`, `src/platform.rs:208`).
- Add a `*_trigger` reader system in `src/ecs/triggers.rs` and register it in `register_triggers` (`src/ecs.rs:163`); for structural entity changes prefer a Bevy observer + `app.add_observer(...)` (`src/ecs.rs:178-185`).

**New domain area (like scroll/focus):**
- Implementation: new `src/ecs/<domain>.rs` exposing `pub struct <Domain>EventsPlugin` with `impl Plugin`.
- Register the plugin in `setup_bevy_app` (`src/ecs.rs:553-558`) and in the test harness (`src/tests/harness.rs`, `setup_world`).

**New CLI/socket command:**
- Variant: `Command`/`Operation` enums in `src/commands.rs:69-135`.
- Parsing: `parse_command` in `src/config.rs` (shared by keybindings and the socket).
- Handler: new system in `src/commands.rs`, registered in `register_commands` (`src/commands.rs:137`) — command systems run in `PreUpdate`.
- Document in `README.md`/`CONFIGURATION.md` if user-facing.

**New OS capability (SkyLight/AX/AppKit):**
- Trait method: `WindowManagerApi` (`src/manager.rs:81`) or the per-object trait (`src/manager/windows.rs:67`, `src/manager/app.rs:57`).
- Implementation: corresponding `*OS` type; new private symbols go in `src/manager/skylight.rs`.
- Mocks: update `src/tests/mocks.rs` so the harness still builds.
- Never call `objc2`/FFI directly from `src/ecs/` — see `AGENTS.md` section 2.

**New configuration option:**
- Field on the right struct in `src/config.rs` or a `src/config/` submodule (`decorations.rs`, `padding.rs`, `swipe.rs` follow this split).
- Accessor on `Config`; consume via `Res<Config>` in systems.
- Document in `CONFIGURATION.md`; deprecated-key warnings hook into `maybe_warn_deprecated_options_for_service` (`src/main.rs:242`).

**New tests:**
- Feature file `src/tests/<feature>.rs` + `mod <feature>;` in `src/tests.rs`.
- Use `TestHarness::new().with_windows(n)` (`src/tests/harness.rs:37`) and `MockState` factories; pure layout math gets plain `#[test]` unit tests colocated in the source file (see `src/platform.rs:308`).

**Utilities:**
- Shared helpers: `src/util.rs` (FFI-adjacent helpers like `AXUIWrapper`, `MacResult`) or the nearest domain module for pure helpers (e.g. `clamp_origin_to_viewport` in `src/ecs/layout.rs:30`).

## Special Directories

**`target/`:**
- Purpose: Cargo build output.
- Generated: Yes.
- Committed: No (`.gitignore`).

**`.planning/`:**
- Purpose: GSD workflow artifacts; `.planning/codebase/` holds these analysis documents.
- Generated: Partially (by GSD tooling).
- Committed: Per project GSD policy.

**`.github/workflows/`:**
- Purpose: CI definitions.
- Generated: No.
- Committed: Yes.

---

*Structure analysis: 2026-07-25*
