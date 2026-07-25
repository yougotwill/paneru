<!-- refreshed: 2026-07-25 -->
# Architecture

**Analysis Date:** 2026-07-25

## System Overview

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        Entry / IPC Layer                             │
├──────────────────────┬──────────────────────────────────────────────┤
│  CLI (clap)          │  Unix socket IPC  `/tmp/paneru.socket`       │
│  `src/main.rs`       │  `src/reader.rs` (CommandReader)             │
└──────────┬───────────┴──────────────────────┬───────────────────────┘
           │                                  │
           ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     Event Bus (mpsc channel)                         │
│   `Event` enum + `EventSender`            `src/events.rs`            │
└──────────┬─────────────────────────────────────────────▲────────────┘
           │                                             │
           ▼                                             │
┌─────────────────────────────────────────────────────────────────────┐
│                  Bevy ECS World (source of truth)                    │
├──────────────────┬──────────────────┬───────────────────────────────┤
│  Schedules       │  Domain plugins  │  Observers / triggers         │
│  `src/ecs.rs`    │  `src/ecs/*.rs`  │  `src/ecs/triggers.rs`        │
│  register_       │  layout,         │  + `src/ecs/restore.rs`       │
│  systems()       │  workspace,      │                               │
│  register_       │  focus, mouse,   │                               │
│  triggers()      │  scroll, display │                               │
└──────────┬───────┴──────────────────┴───────────────┬───────────────┘
           │                                          │
           ▼                                          │
┌─────────────────────────────────────────────────────────────────────┐
│              OS Bridge (main-thread, NonSend resources)              │
├──────────────────────┬──────────────────────────────────────────────┤
│  WindowManagerApi    │  PlatformCallbacks (event sources)           │
│  `src/manager.rs`    │  `src/platform.rs` + `src/platform/`         │
│  `src/manager/`      │  input / process / workspace / display /     │
│  (WindowOS,          │  mission_control / notify / service          │
│   ApplicationOS,     │                                              │
│   skylight FFI)      │  UI chrome: `src/menubar.rs`,                │
│                      │  `src/overlay.rs`                            │
└──────────┬───────────┴──────────────────────────────────────────────┘
           │ AX / SkyLight / CoreGraphics FFI
           ▼
┌─────────────────────────────────────────────────────────────────────┐
│              macOS WindowServer / AppKit / Accessibility             │
└─────────────────────────────────────────────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| CLI & daemon bootstrap | clap subcommands (`launch`, `install`, `send-cmd`, `query`, `subscribe`), accessibility gate, logging | `src/main.rs` |
| Bevy app assembly | Builds `BevyApp`, inserts resources/NonSend resources, adds plugins | `src/ecs.rs` (`setup_bevy_app`, line 535) |
| System registration | Schedule wiring for `Startup`/`PreUpdate`/`Update`/`PostUpdate` | `src/ecs.rs` (`register_systems`, line 59) |
| Trigger registration | Message-reading triggers + Bevy observers | `src/ecs.rs` (`register_triggers`, line 163) |
| Marker & state components | `FocusedMarker`, `ActiveWorkspaceMarker`, `Unmanaged`, `RepositionMarker`, etc. | `src/ecs.rs` (lines 188–446) |
| Core systems | Event pump, lifecycle (add process/app/window), animation, FFI commit, cleanup | `src/ecs/systems.rs` |
| Reactive handlers | Observers for spawn/destroy/focus/minimize/config-refresh events | `src/ecs/triggers.rs` |
| Tiling layout | `LayoutStrip`, `Column`, `StackItem`; strip positioning math | `src/ecs/layout.rs` |
| Workspaces | Virtual strips per macOS space, workspace switching/reaping | `src/ecs/workspace.rs` |
| Focus | `FocusHistory`, focus-follows-mouse, tier memory (floating vs tiled) | `src/ecs/focus.rs` |
| Mouse/scroll input | Drag/swipe gesture systems | `src/ecs/mouse.rs`, `src/ecs/scroll.rs` |
| Displays | Display reconcile, `FloatingLayer` tier, dock position | `src/ecs/display.rs` |
| Session state | `PaneruState` snapshot, save/load, state queries | `src/ecs/state.rs` |
| Session restore | `SessionRestore` grace-period resource, restore planning | `src/ecs/restore.rs` |
| System params | `Windows`, `ActiveDisplay`, `ActiveDisplayMut`, `GlobalState` | `src/ecs/params.rs` |
| OS abstraction traits | `WindowManagerApi`, `WindowApi`, `ApplicationApi`, `ProcessApi` (all `#[automock]`) | `src/manager.rs`, `src/manager/*.rs` |
| macOS implementations | `WindowManagerOS`, `WindowOS`, `ApplicationOS` — SkyLight + AX FFI | `src/manager.rs`, `src/manager/windows.rs`, `src/manager/app.rs` |
| SkyLight private API | `extern` declarations for `SLS*`/`_AX*` functions | `src/manager/skylight.rs` |
| Platform event sources | `PlatformCallbacks` aggregating CGEventTap, Carbon PSN events, NSWorkspace observers, display reconfiguration, Mission Control AX observer, distributed notifications | `src/platform.rs`, `src/platform/` |
| launchd service | install/uninstall/start/stop/restart via `launchctl` | `src/platform/service.rs` |
| Command handlers | `Command`/`Operation` enums + PreUpdate handler systems | `src/commands.rs` |
| State query IPC | JSON state/virtual-workspaces/active responses | `src/commands/query.rs` |
| Configuration | TOML parsing, hot reload, bindings, window rules | `src/config.rs`, `src/config/` |
| Menu bar item | `NSStatusItem` menu (incl. accessibility-required mode) | `src/menubar.rs` |
| Overlays | Border highlight, inactive dimming, flash messages (NSWindow) | `src/overlay.rs` |
| Event definitions | `Event` enum, `EventSender` mpsc wrapper | `src/events.rs` |
| Error type | `Error` enum + `Result<T>` alias | `src/errors.rs` |
| IPC socket | `CommandReader` Unix socket server/client | `src/reader.rs` |
| Test harness | `TestHarness`, `MockState`, mock WM/process/window | `src/tests/harness.rs`, `src/tests/mocks.rs` |

## Pattern Overview

**Overall:** Data-driven Entity Component System (Bevy 0.18, `MinimalPlugins`) acting as the single source of truth, bridged to macOS via a main-thread event pump and trait-object FFI wrappers.

**Key Characteristics:**

- **ECS as source of truth.** Tiling logic operates only on components (`LayoutStrip`, `WidthRatio`, `Position`). The physical macOS window state is a *reflection* of ECS state, written out by commit systems in `PostUpdate`.
- **Marker components over state machines.** Filtering/state tracking is done with zero-sized markers: `FocusedMarker`, `ActiveWorkspaceMarker`, `ActiveDisplayMarker`, `FreshMarker`, `ExistingMarker`, `NativeFullscreenMarker`, `SelectedVirtualMarker` (all in `src/ecs.rs`). Managed-state is an enum component: `Unmanaged::{Floating, Minimized, Hidden}` (`src/ecs.rs:285`).
- **Command-pattern via marker insertion.** Systems never move windows directly. They insert `RepositionMarker(Origin)` / `ResizeMarker(Size)` (usually via the `SpawnCommandsExt` helper, `src/ecs.rs:448`), and `PostUpdate` commit systems perform the FFI call. Example (`src/ecs.rs:471`):

  ```rust
  fn reposition_entity(&mut self, entity: Entity, origin: Origin) {
      if let Ok(mut entity_commands) = self.get_entity(entity) {
          entity_commands.try_insert(RepositionMarker(origin));
      }
  }
  ```
- **Trigger/observer reactivity.** OS events arrive as Bevy `Messages<Event>` read by `*_trigger` systems in `Update` (`src/ecs/triggers.rs`), while structural changes fire Bevy observers (`window_managed_trigger`, `spawn_window_trigger`, `restore_window_state`, …) registered via `app.add_observer(...)` (`src/ecs.rs:178-185`).
- **Change detection gates FFI.** `Changed<LayoutStrip>`, `Added<FocusedMarker>`, `RemovedComponents<FocusedMarker>` etc. drive expensive work — e.g. the `overlay_dirty` run condition (`src/ecs.rs:77-86`).
- **Entity hierarchy mirrors OS hierarchy.** Process → Application → Window via `ChildOf`; Display → LayoutStrip via `ChildOf` (`src/ecs/systems.rs:56-82`).

## Layers

**Entry / IPC Layer:**
- Purpose: Parse CLI, gate on accessibility permission, run daemon or forward commands.
- Location: `src/main.rs`, `src/reader.rs`, `src/accessibility_prompt.rs`
- Contains: clap `Parser`/`Subcommand` structs, `wait_for_accessibility` loop (`src/main.rs:186`), Unix-socket `CommandReader` (`src/reader.rs:17`)
- Depends on: `events`, `config`, `platform::service`, `ecs::setup_bevy_app`
- Used by: end user / external scripts (`paneru send-cmd`, `paneru query`, `paneru subscribe`)

**Event Bus:**
- Purpose: Decouple producer threads (OS callbacks, socket reader, config watcher) from the single-threaded ECS world.
- Location: `src/events.rs`
- Contains: `Event` enum (`src/events.rs:20`) — one variant per OS occurrence (`WindowCreated`, `ApplicationFrontSwitched`, `SpaceChanged`, `Swipe`, `ConfigRefresh`, `Command`, `StateQuery`, …); `EventSender` (`src/events.rs:167`) wrapping `std::sync::mpsc::Sender`.
- Depends on: nothing (leaf module)
- Used by: every platform handler, the socket reader, and `pump_events`

**ECS Core:**
- Purpose: Own all window-management state and behavior.
- Location: `src/ecs.rs`, `src/ecs/`
- Contains: components, resources, systems, triggers, domain plugins
- Depends on: `manager` (trait objects only), `events`, `config`, `commands`, `platform` types (`WinID`, `WorkspaceId`, `Modifiers`)
- Used by: `main` (via `setup_bevy_app`)

**OS Bridge (`manager`):**
- Purpose: Present a mockable, (mostly) OS-agnostic API over macOS windowing.
- Location: `src/manager.rs`, `src/manager/`
- Contains: `WindowManagerApi` trait (`src/manager.rs:81`), Bevy resource `WindowManager(pub Box<dyn WindowManagerApi>)` (`src/manager.rs:192`), macOS impl `WindowManagerOS` (`src/manager.rs:196`), per-object traits `WindowApi` (`src/manager/windows.rs:67`) / `ApplicationApi` (`src/manager/app.rs:57`) / `ProcessApi` (`src/manager/process.rs`) wrapped in Bevy components `Window`, `Application`, `BProcess`; private-SkyLight FFI in `src/manager/skylight.rs`.
- Depends on: `platform` primitives, `util` (`AXUIWrapper`, `MacResult`), `errors`
- Used by: ECS systems (`Res<WindowManager>`, `Query<&Window>`, …)

**Platform Layer:**
- Purpose: Subscribe to every macOS event source and normalize into `Event`s; own the Cocoa run loop.
- Location: `src/platform.rs`, `src/platform/`
- Contains: `PlatformCallbacks` (`src/platform.rs:140`) with `main_thread_marker: MainThreadMarker`; handlers `input.rs` (CGEventTap on main CFRunLoop), `process.rs` (Carbon PSN), `workspace.rs` (`NSWorkspace` observers), `display.rs` (CGDisplay reconfiguration), `mission_control.rs` (AXObserver on Dock), `notify.rs` (distributed notifications: dock, theme, wake), `service.rs` (launchd).
- Depends on: `events`, `config`, `manager::{check_ax_privilege, check_separate_spaces}`
- Used by: `setup_bevy_app` (inserted as NonSend resource)

**UI Chrome:**
- Purpose: Native overlays that must live on the main thread.
- Location: `src/menubar.rs`, `src/overlay.rs`
- Contains: `MenuBarManager` (`src/menubar.rs:99`), `OverlayManager` + `FlashMessageManager` (`src/overlay.rs:240`, `:444`) — raw `NSWindow` borders/dimming.
- Depends on: `objc2_app_kit`, `MainThreadMarker`
- Used by: ECS systems via `NonSend`/`NonSendMut`

## Data Flow

### Primary Request Path (OS event → ECS mutation → OS write-back)

1. A macOS callback fires (e.g. AX "window created", CGEventTap swipe) in a platform handler (`src/platform/*.rs`) and is forwarded as `Event` through `EventSender` (`src/events.rs:192`).
2. `systems::pump_events` (`src/ecs/systems.rs:584`, `PreUpdate`) pumps the Cocoa run loop (`PlatformCallbacks::pump_cocoa_event_loop`, `src/platform.rs:237`), drains the mpsc `Receiver<Event>` (NonSend), coalesces `MouseMoved`, and writes a batch of Bevy `Messages<Event>`. It also adapts the frame timeout: 16 ms while animating, 50 ms idle, 500 ms in low-power mode.
3. Trigger systems in `Update` (`src/ecs/triggers.rs`, registered at `src/ecs.rs:163`) read the messages and mutate the world — e.g. `window_focused_trigger` moves `FocusedMarker`, `front_switched_trigger` re-triggers `SendMessageTrigger(Event::WindowFocused …)`. Observers (`spawn_window_trigger`, `window_managed_trigger`, …) react to structural `Trigger`s.
4. Domain plugins recompute layout: `LayoutEventsPlugin` chains `sync_tab_group_frames → layout_sizes_changed → layout_strip_changed → reshuffle_layout_strip → ensure_visible_in_strip → position_layout_strips → position_layout_windows` (`src/ecs/layout.rs:39-58`), inserting `RepositionMarker`/`ResizeMarker` on windows.
5. `PostUpdate` commit chain `animate_entities → commit_window_position → verify_window_position` (`src/ecs.rs:137-142`; impls at `src/ecs/systems.rs:497`, `:932`, `:942`) consumes the markers and calls `window.reposition(origin)` / `window.resize(size)` on the `Window` trait object, which performs the AX/SkyLight FFI (`src/manager/windows.rs`).

### Command Path (CLI → daemon)

1. `paneru send-cmd …` serializes argv to the socket (`CommandReader::send_command`, `src/reader.rs:35`).
2. Daemon-side `CommandReader::runner` (`src/reader.rs:104`) parses with `config::parse_command` and sends `Event::Command`.
3. `register_commands` (`src/commands.rs:137`) installs `PreUpdate` handler systems (`resize_window`, `command_center_window`, `focus_window`, …) that consume the message and mutate ECS state; layout/commit then proceeds as in the primary path.

### Query/Subscribe Path

1. `paneru query state --json` → socket → `Event::StateQuery { kind, respond_to }` (`src/reader.rs:130`).
2. `commands::query::register_query_commands` (`src/commands/query.rs:160`) systems build the JSON from `PaneruState`/live ECS and reply on the oneshot channel (2 s timeout in the reader).
3. `paneru subscribe` registers the socket stream via `Event::StateSubscribe`; state broadcasts are pushed as line-delimited JSON (see `QUERY_AND_SUBSCRIBE_FORMAT.md`).

### Session Restore Path

1. `setup_bevy_app` loads `PaneruState` from `~/.local/state/paneru/state.json` and inserts it as a resource (`src/ecs.rs:574-578`).
2. The `restore_window_state` observer (`src/ecs/restore.rs`, registered `src/ecs.rs:185`) matches arriving windows against saved identities (hard key: `window_id`+`pid`+`bundle_id`; conservative fallback heuristics) during the `SessionRestore` grace timer.
3. `tick_restore_grace` (`src/ecs/restore.rs:47`, `Update`) removes `SessionRestore`/`PaneruState` when the grace period ends; afterwards normal config window-rules own placement.
4. `state::periodic_state_save` (`src/ecs.rs:130`) rewrites the snapshot every 300 s; `state::cleanup_on_exit` writes a final snapshot.

**State Management:**
- All mutable state lives in the Bevy `World` (components/resources). Global flags are resources: `SkipReshuffle`, `MissionControlActive`, `FocusFollowsMouse`, `Initializing`, `LowPowerMode`, `SystemTheme` (`src/ecs.rs:407-433`).
- Systems access common query shapes through `SystemParam` bundles (`src/ecs/params.rs`) instead of hand-writing queries: `Windows` (all/focused/positional queries), `ActiveDisplay`/`ActiveDisplayMut` (single active display + strip), `GlobalState` (FFM flag, skip-reshuffle, initializing).

## Key Abstractions

**`WindowManagerApi` trait + `WindowManager` resource:**
- Purpose: Mockable facade over the macOS windowing system (displays, spaces, window lists, mouse warp, dimming, config watcher).
- Examples: trait at `src/manager.rs:81`; resource wrapper `src/manager.rs:192`; macOS impl `WindowManagerOS` at `src/manager.rs:196`; construction in `setup_bevy_app` (`src/ecs.rs:536`).
- Pattern: `#[automock]` (mockall) — tests inject `MockWindowManagerApi` built by `MockState` (`src/tests/mocks.rs`). Always program against the trait, never against `WindowManagerOS`.

**`WindowApi` / `ApplicationApi` / `ProcessApi` trait objects as components:**
- Purpose: Per-entity OS handles with dynamic dispatch, stored directly as Bevy components: `Window(Box<dyn WindowApi>)` (`src/manager/windows.rs:100`), `Application` (`src/manager/app.rs`), `BProcess(Box<dyn ProcessApi>)` (`src/ecs.rs:303`).
- Pattern: `Deref`/`DerefMut` via `derive_more` so systems call `window.id()`, `window.reposition(..)` directly.

**`LayoutStrip` / `Column` / `StackItem`:**
- Purpose: The core tiling data structure — a horizontal strip of columns per (macOS space, virtual index); each column is `Single`, `Stack(Vec<StackItem>)`, `Tabs(Vec<Entity>)`, or `Fullscren`.
- Examples: `LayoutStrip` at `src/ecs/layout.rs:239`; spawned only through `SpawnCommandsExt::spawn_layout_strip` (`src/ecs.rs:518`) which attaches `Position` + `ChildOf(display)` + `ActiveWorkspaceMarker`/`SelectedVirtualMarker`.
- Pattern: pure data + pure functions (`clamp_origin_to_viewport`, `src/ecs/layout.rs:30`) — no OS calls inside layout math.

**SystemParam bundles (`Windows`, `ActiveDisplay`, `GlobalState`):**
- Purpose: Encode the canonical query shapes and invariants (exactly one active display/strip via `Single`).
- Examples: `src/ecs/params.rs:84` (`ActiveDisplay`), `:202` (`Windows`), `:29` (`GlobalState`).

**`Event` / `EventSender`:**
- Purpose: Single normalized event type crossing all thread boundaries.
- Example: `src/events.rs:20`.

**`Origin` / `Size`:**
- Purpose: Coordinate types — both are aliases of `bevy::math::IVec2` (`src/manager.rs:59-60`), with `origin_from`/`origin_to`/`size_from` converters bridging `CGPoint`/`CGSize` (macOS Y-down) into the ECS.

**`Timeout` component:**
- Purpose: Deferred/one-shot system execution (`SystemId`) on a timer — retries for stray focus events, flash message expiry (`src/ecs.rs:306-351`).

## Entry Points

**`main()` (CLI + daemon):**
- Location: `src/main.rs:125`
- Triggers: process launch (user or launchd)
- Responsibilities: init `tracing` (stderr, `EnvFilter`, default `info`); parse `SubCmd`; for `Launch`: set ctrl-c handler → `Event::Exit`, start `CommandReader` thread, gate on accessibility (`wait_for_accessibility`, `src/main.rs:186`), then `setup_bevy_app(sender, receiver)` and `app.run()`. Other subcommands drive `service::Service` (launchd) or `CommandReader` socket requests.

**`setup_bevy_app`:**
- Location: `src/ecs.rs:535`
- Triggers: `SubCmd::Launch`
- Responsibilities: create `WindowManagerOS`, start config watcher, build `BevyApp` with `MinimalPlugins`, clamp `Time::<Virtual>` max delta to 10 s, insert resources (`WindowManager`, `SkipReshuffle`, `SystemTheme`, `MissionControlActive`, `FocusFollowsMouse`, `Initializing`), NonSend resources (config watcher, `PlatformCallbacks`, `OverlayManager`, `FlashMessageManager`, `MenuBarManager`, event `Receiver`), add domain plugins + `register_triggers` + `register_systems` + `register_commands`, load `PaneruState`.

**Schedule wiring (`register_systems`):**
- Location: `src/ecs.rs:59`
- Key ordering:
  - `Startup`: `gather_displays → gather_initial_processes` (chained).
  - `PreUpdate`: `window_creation_event`, `pump_events`.
  - `Update`: window defaults/positions chain; init chain (`add_existing_process → add_existing_application → finish_setup`, gated on `resource_exists::<Initializing>`); retry/timeout tickers; move/resize frame sync (gated by `not_swiping`); restore tick; 300 s state save.
  - `PostUpdate`: `animate_entities → commit_window_position → verify_window_position`; `animate_resize_entities → commit_window_size`; overlay updates (gated by `dimming_enabled` + `overlay_dirty`); `menubar::update_menu_bar`.
- Run-condition closures (`not_swiping`, `dimming_enabled`, `overlay_dirty`, `native_tabs_enabled`) are defined at `src/ecs.rs:62-88` — copy this pattern for new conditional systems.

**Domain plugins:** each `src/ecs/<domain>.rs` exposes a `Plugin` (`LayoutEventsPlugin`, `WorkspaceEventsPlugin`, `FocusEventsPlugin`, `MouseEventsPlugin`, `ScrollEventsPlugin`, `DisplayEventsPlugin`) that registers its own systems/observers. New domain areas should follow this pattern and be added in `setup_bevy_app` (`src/ecs.rs:553-558`).

## Architectural Constraints

- **Threading:** The Bevy app runs on the main thread; `pump_events` manually drives the Cocoa loop each frame (`src/ecs/systems.rs:601`). All AppKit/CoreGraphics/AX calls MUST happen on the main thread — enforced by holding `PlatformCallbacks`, `OverlayManager`, `FlashMessageManager`, `MenuBarManager`, the config watcher, and the event `Receiver` as **NonSend** resources (`src/ecs.rs:552`, `:568-572`), which pins the systems using them to the main thread. `MainThreadMarker` (`src/platform.rs:175`) is threaded into UI chrome constructors. `bevy` is built with `multi_threaded`, so CPU-heavy pure work (e.g. `bruteforce_windows`, `src/manager.rs:726`) is offloaded via `AsyncComputeTaskPool` + `Task` components (`BruteforceWindows`, `src/ecs.rs:363`).
- **Global state:** No global mutable state beyond: `CONFIGURATION_FILE: LazyLock<PathBuf>` (`src/config.rs:42`), `AX_NOTIFICATIONS`/`AX_WINDOW_NOTIFICATIONS: LazyLock` (`src/manager/app.rs:32`,`:48`), `ENHANCED_UI_REFCOUNT: LazyLock<Mutex<..>>` (`src/manager/windows.rs:38`), cached `macos_major_version: OnceLock` (`src/platform.rs:300`). Everything else is ECS resources.
- **Accessibility privilege:** Hard gate before the app starts — `check_ax_privilege` (`src/manager.rs:792`) and "Displays have separate Spaces" (`check_separate_spaces`, `src/manager.rs:818`) are verified in `PlatformCallbacks::setup_handlers` (`src/platform.rs:208`).
- **ECS-first invariant:** never read back OS window state to make layout decisions; the world is authoritative. OS reads happen only to (a) ingest new facts as `Event`s and (b) verify writes (`verify_window_position`, retry-bounded by `VerifyWindowPosition { remaining: 3 }`, `src/ecs.rs:390`).
- **Initialization gating:** Systems that must not run during world bootstrap use `run_if(resource_exists::<Initializing>)` / `not(resource_exists::<Initializing>)`; `finish_setup` (`src/ecs/systems.rs:176`) removes the `Initializing` resource.
- **Circular imports:** none known; module layering is strictly `platform`/`events`/`errors` (leaves) ← `manager` ← `config` ← `commands` ← `ecs` ← `main`.

## Anti-Patterns

### Calling `objc2`/SkyLight/AX directly from an ECS system

**What happens:** A system imports `objc2_app_kit` or calls an `SLS*` function directly.
**Why it's wrong:** Bypasses the mockable seam — the test harness (`src/tests/harness.rs`) cannot intercept it, and main-thread affinity becomes unverifiable.
**Do this instead:** Add a method to `WindowManagerApi` (`src/manager.rs:81`) or the relevant `*Api` trait, implement it in the `*OS` type, and update the mocks in `src/tests/mocks.rs`.

### Mutating `Position`/`Bounds` to move a window

**What happens:** A system writes `position.0 = new_origin` expecting the OS window to follow.
**Why it's wrong:** Only `RepositionMarker`/`ResizeMarker` consumption in `PostUpdate` performs the FFI write; direct mutation desynchronizes ECS from the real window (and `verify_window_position` will fight you).
**Do this instead:** Use `commands.reposition_entity(entity, origin)` / `resize_entity` / `reshuffle_around` / `focus_entity` from `SpawnCommandsExt` (`src/ecs.rs:448`).

### OS calls inside layout math

**What happens:** `src/ecs/layout.rs` functions query `WindowManager` or AX attributes while computing frames.
**Why it's wrong:** Layout must stay pure and unit-testable; OS reads also break the ECS-as-source-of-truth invariant.
**Do this instead:** Pass needed facts in as parameters (`IRect` bounds, `&Config`) like `clamp_origin_to_viewport` (`src/ecs/layout.rs:30`).

### `unwrap()` / panics in systems

**What happens:** A transient AX error (`kAXErrorCannotComplete`) crashes the daemon.
**Why it's wrong:** macOS windowing APIs fail transiently all the time.
**Do this instead:** Return/ propagate `Result` (`src/errors.rs`), log with `inspect_err(|err| warn!(...))`, and schedule a retry via `Timeout` + marker (`StrayFocusEvent`, `RetryFrontSwitch` — see `front_switched_trigger`, `src/ecs/triggers.rs:106-118`).

### Polling systems without run conditions

**What happens:** A system runs every frame regardless of need, defeating the reactive idle loop.
**Why it's wrong:** `pump_events` scales frame latency (16/50/500 ms) assuming quiescent frames stay cheap.
**Do this instead:** Gate with `Changed<T>` queries, `resource_exists`, `on_timer`, or a custom run-condition closure like `overlay_dirty` (`src/ecs.rs:77`).

## Error Handling

**Strategy:** Single crate-wide `Error` enum + `Result<T>` alias (`src/errors.rs:5-26`); blanket `From<T: std::error::Error + Display>` converts I/O and `notify` errors.

**Patterns:**
- Systems log-and-continue: `window_manager.active_display_id().inspect_err(...)` / early-return (`src/ecs/systems.rs:57-60`).
- Retry-with-timeout for transient AX failures: `Timeout` component + marker entities (`RetryFrontSwitch`, `StrayFocusEvent`), despawned by `timeout_ticker`/`cleanup_timeout_trigger`.
- FFI status codes wrapped by `MacResult::to_result(function_name!())` (`src/util.rs`) so errors carry the calling function name.
- Fatal startup errors (no AX privilege, no separate spaces) abort `setup_handlers` with `Error::PermissionDenied`/`Error::InvalidConfig` and `main` stops the launchd service (`src/main.rs:162-167`).

## Cross-Cutting Concerns

**Logging:** `tracing` + `tracing-subscriber` (compact, stderr, file/line/target, `EnvFilter`). Complex systems use `#[instrument(level = Level::DEBUG, skip_all, fields(...))]` — see `window_focused_trigger` (`src/ecs/triggers.rs:177`) and `SpawnCommandsExt` (`src/ecs.rs:472`). Use `trace` for per-frame hot paths (`get_associated_windows`, `src/manager.rs:327`).

**Validation:** Configuration validated at load in `Config::new` (`src/config.rs`); window rules evaluated through `WindowProperties` (`src/ecs.rs:586`) — never re-parse TOML in systems; read `Res<Config>` (hot-reloaded by `refresh_configuration_trigger`, `src/ecs/triggers.rs`).

**Authentication/permissions:** macOS Accessibility entitlement only; checked via `check_ax_privilege`/`request_ax_privilege` (`src/manager.rs:792`,`:800`); guided setup UI in `src/accessibility_prompt.rs`.

**Persistence:** `PaneruState` JSON snapshot (version field `SUPPORTED_STATE_VERSION = 2`, `src/ecs/state.rs:25`) written atomically to the XDG state dir every 300 s and on exit.

---

*Architecture analysis: 2026-07-25*
