# Testing Patterns

**Analysis Date:** 2026-07-25

## Test Framework

**Runner:**
- Built-in Rust test harness via `cargo test` — no external test runner (no nextest, no custom harness).
- No `[dev-dependencies]` section: `mockall = "0.14"` is a **main dependency** (`Cargo.toml:75`) because `#[automock]` is applied directly to the production API traits, so the `Mock*` types are always compiled (mocks are re-exported to consumers behind `#[cfg(test)]`, e.g. `src/manager.rs:48-51`).
- Toolchain: stable, pinned by `rust-toolchain.toml`; edition 2024 (`Cargo.toml:4`).
- The test tree is gated in `src/main.rs:22-23`:
  ```rust
  #[cfg(test)]
  mod tests;
  ```

**Assertion Library:**
- Standard `assert!`/`assert_eq!` plus custom `#[macro_export]` assertion macros in `src/tests/harness.rs` (see below).

**Run Commands:**
```bash
cargo test                          # Run all tests
cargo test test_window_shuffle      # Run one test by name
cargo test tests::tiling            # Run a module (tiling integration tests)
cargo test -- --nocapture           # Show tracing output (tests log to stderr)
RUST_LOG=debug cargo test           # Verbose tracing (harness installs EnvFilter, default "info")
cargo clippy && cargo fmt --check   # Lint/format gates (per AGENTS.md)
```

Note: CI (`.github/workflows/rust.yml`) currently only runs `cargo build --release` on `macos-latest` — tests and clippy are run locally, not in CI.

## Test File Organization

**Location:**
- Two styles, both used:
  1. **ECS integration tests** in a dedicated tree: `src/tests.rs` (module root) + `src/tests/*.rs`.
  2. **Inline unit tests** in `#[cfg(test)] mod tests` at the bottom of the source file under test — `src/ecs/layout.rs:1399` (25 tests), `src/config.rs:1674` (17 tests), `src/commands.rs:1460`, `src/commands/query.rs:215,414`, `src/menubar.rs:346`, `src/platform/input.rs:500`, `src/ecs/mouse.rs:470`, `src/ecs/focus.rs:358`, `src/manager/app.rs:633`, `src/manager/windows.rs:734`, `src/platform.rs:308`.

**`src/tests.rs` root** declares the submodules and shared constants:
```rust
mod display; mod harness; mod interaction; mod mocks;
mod session_restore; mod state; mod tabs; mod tiling;

pub(crate) use harness::*;
pub(crate) use mocks::*;

pub(crate) const TEST_PROCESS_ID: i32 = 1;
pub(crate) const TEST_DISPLAY_ID: u32 = 1;
pub(crate) const TEST_WORKSPACE_ID: u64 = 2;
pub(crate) const TEST_DISPLAY_WIDTH: i32 = 1024;
pub(crate) const TEST_DISPLAY_HEIGHT: i32 = 768;
pub(crate) const EXT_DISPLAY_ID: u32 = 2;      // second display fixtures
pub(crate) const EXT_WORKSPACE_ID: u64 = 20;
pub(crate) const TEST_MENUBAR_HEIGHT: i32 = 20;
pub(crate) const TEST_WINDOW_WIDTH: i32 = 400;
pub(crate) const TEST_WINDOW_HEIGHT: i32 = 1000;
```
Always use these constants instead of magic numbers in new tests.

**Structure:**
```
src/tests.rs            # module root, constants, re-exports
src/tests/
├── harness.rs          # TestHarness builder + assert_*! macros (335 lines)
├── mocks.rs            # MockState "virtual macOS" + mockall factories (696 lines)
├── tiling.rs           # window shuffle/balance/startup tiling (5 tests)
├── tabs.rs             # native tab detection/resize sync (7 tests)
├── state.rs            # PaneruState serialization/restoration (16 tests)
├── session_restore.rs  # session restore flows (12 tests)
├── interaction.rs      # scroll/swipe, fullscreen, focus, drag (34 tests)
└── display.rs          # multi-display behavior (9 tests)
```

## Test Structure

**Suite Organization (integration style):**
Tests drive a real Bevy `App` (with `MinimalPlugins` + all of Paneru's real systems/triggers) against a mocked OS layer. The canonical pattern from `src/tests/tiling.rs:111-136`:

```rust
#[test]
fn test_window_balance() {
    let commands = vec![
        Event::MenuOpened { window_id: 0 },
        Event::Command {
            command: Command::Window(Operation::Resize(ResizeDirection::Grow)),
        },
        Event::Command {
            command: Command::Window(Operation::Balance),
        },
    ];

    TestHarness::new()
        .with_windows(3)
        .on_iteration(1, |world, _state| {
            // After grow, window 0 should be 512 (50% of 1024).
            assert_window_size!(world, 0, 512, 748);
        })
        .on_iteration(2, |world, _state| {
            assert_window_size!(world, 0, 512, 748);
            assert_window_size!(world, 1, 512, 748);
            assert_window_size!(world, 2, 512, 748);
        })
        .run(commands);
}
```

**Patterns:**
- **Setup:** `TestHarness::new()` builds the world with one default display (1024×768), one workspace, one test app (pid 1), and a mocked `WindowManager` resource. Builder methods: `.with_windows(n)`, `.with_window(id, |data| ...)`, `.with_workspace_window(id, ws, f)`, `.with_display(id, bounds, workspaces)`, `.with_config(config)`, `.with_state(PaneruState)`, `.with_app(pid, bundle, name, f)`, `.with_focused_window(id)` (`src/tests/harness.rs:37-167`).
- **Execution:** `.run(commands: Vec<Event>)` (`src/tests/harness.rs:177-194`) — each `Event` is one "iteration": the event is written via `write_message::<Event>()`, then **5 `app.update()` ticks** run, draining any events the mock OS generated back into the world after each tick. This models the OS↔WM feedback loop.
- **Verification:** `.on_iteration(i, |world, mock_state| ...)` registers a closure run after iteration `i`. Inside, query the world directly (`world.query::<&LayoutStrip>()`), mutate it (`world.entity_mut(e).remove::<FocusedMarker>()`), or drive the mock OS (`state.update_window(...)`, `state.activate_workspace(...)`) — see `src/tests/interaction.rs:45-59` and the helper `spawn_matching_native_tab` in `src/tests/tabs.rs:13-25`.
- **Teardown:** none needed — each test builds a fresh `App`/`MockState`. Global one-time init (tracing subscriber, single-threaded `AsyncComputeTaskPool`) is guarded by a `OnceLock` in `setup_world()` (`src/tests/harness.rs:197-243`).
- **Time control:** `TimeUpdateStrategy::ManualDuration(Duration::from_millis(100))` (`src/tests/harness.rs:238-240`) makes Bevy timers deterministic.
- **World construction:** the harness registers the *real* plugins — `MouseEventsPlugin`, `ScrollEventsPlugin`, `WorkspaceEventsPlugin`, `LayoutEventsPlugin`, `FocusEventsPlugin`, `DisplayEventsPlugin`, plus `(register_triggers, register_systems, register_commands)` (`src/tests/harness.rs:221-236`). Tests exercise the production system graph, not a stub.

**Inline unit test style** (pure logic, no app) from `src/ecs/layout.rs:1399-1444`:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use bevy::prelude::*;

    #[test]
    fn clamp_origin_supports_oversized_windows() {
        let viewport = IRect::new(0, 20, 1024, 768);
        let size = Size::new(2048, 748);
        assert_eq!(
            clamp_origin_to_viewport(Origin::new(300, 20), size, viewport),
            Origin::new(0, 20)
        );
    }

    fn setup_world_and_strip() -> (World, LayoutStrip, Vec<Entity>) {
        let mut world = World::new();
        let entities = world.spawn_batch(vec![(), (), ()]).collect::<Vec<Entity>>();
        let mut strip = LayoutStrip::default();
        strip.append(entities[0]);
        // ...
        (world, strip, entities)
    }
}
```
Follow this for layout math and `LayoutStrip` operations (append/stack/swap/tabs) — a bare `World` with `spawn_batch` is enough; no harness required.

## Mocking

**Framework:** `mockall` — `#[automock]` on the four OS-boundary traits:
- `WindowManagerApi` → `MockWindowManagerApi` (`src/manager.rs:80`)
- `WindowApi` → `MockWindowApi` (`src/manager/windows.rs:66`)
- `ApplicationApi` → `MockApplicationApi` (`src/manager/app.rs:56`)
- `ProcessApi` → `MockProcessApi` (`src/manager/process.rs:72`)

**Pattern — "Virtual macOS":** rather than hand-setting expectations per test, `MockState` (`src/tests/mocks.rs:90-108`) is a shared fake OS:

```rust
#[derive(Clone)]
pub struct MockState {
    inner: Arc<RwLock<MockStateInner>>,   // apps, windows, displays, event_queue
}
```

Factory methods wire `mockall` expectations to closures that read/mutate this state (`src/tests/mocks.rs:344-696`):

```rust
pub fn create_window(&self, id: WinID) -> Window {
    let mut mw = MockWindowApi::new();
    mw.expect_id().return_const(id);
    let s = self.clone();
    mw.expect_resize().returning(move |size| {
        let mut inner = s.inner.force_write();
        if let Some(w) = inner.windows.get_mut(&id) {
            w.frame.max = w.frame.min + size;
        }
    });
    // ... every accessor reads from MockStateInner
    Window::new(Box::new(mw))
}
```

Key consequences to preserve when extending mocks:
- WM actions (resize/reposition/focus) **mutate the virtual OS**, so subsequent reads see the new frame — this is what makes multi-tick assertions meaningful.
- OS-originated changes go through behavior methods that also **enqueue OS events**: `focus_window` pushes `Event::ApplicationFrontSwitched` + `Event::WindowFocused`; `os_move_window` pushes `Event::WindowMoved`; `os_minimize_window` pushes `Event::WindowMinimized` (`src/tests/mocks.rs:158-291`). The harness drains these via `drain_events()` each tick.
- Interaction helpers: `simulate_click`, `simulate_window_click`, `simulate_drag` enqueue mouse events (`src/tests/mocks.rs:295-336`).
- Locking uses `stdext::prelude::RwLockExt` — `force_write()`/`force_read()` (poison-resistant), never bare `.write().unwrap()`.
- Fixtures: `MockWindowData::default()` (`src/tests/mocks.rs:50-70`) provides AX role `"AXWindow"`/subrole `"AXStandardWindow"`, visible, non-minimized; override per-test with `update_window(id, |w| w.is_full_screen = true)` style closures.
- Fill remaining `expect_*` with `return_const` defaults (e.g. `mw.expect_element().return_const(None);` at `src/tests/mocks.rs:500-503`).

**What to Mock:**
- Only the OS boundary: `WindowManagerApi`, `WindowApi`, `ApplicationApi`, `ProcessApi`. Add new OS-facing trait methods to `src/tests/mocks.rs` factories in the same closure-over-shared-state style.

**What NOT to Mock:**
- ECS internals: real `LayoutStrip`, real systems/triggers, real `Config`, real `Event` dispatch. Don't stub `Window` component data separately from `MockState` — the component and the mock must agree, so always spawn windows via `MockState::spawn_window` + `SpawnWindowTrigger`.

## Fixtures and Factories

**Test Data:**
- Constants from `src/tests.rs` (see above). `EXT_*` constants define the second display (1920×1200) for multi-display tests in `src/tests/display.rs`.
- Serialized-state fixtures are built inline as `PaneruState { version: 2, displays: vec![SavedDisplay { ... }], ... }` structs — see `src/tests/state.rs:60-103`, which round-trips through `serde_json` and asserts equality.

**Location:**
- No separate fixtures directory; data lives in constants, `Default` impls (`MockWindowData`), and per-test struct literals.

**Helper functions/macros** (all `#[macro_export]`, defined in `src/tests/harness.rs:253-335`):
- `find_window_entity(window_id, world) -> Entity` (`src/tests/harness.rs:245-251`) — panics with a clear message if missing.
- `assert_window_at!(world, id, x, y)` — asserts `window.frame().min`.
- `assert_window_size!(world, id, w, h)`.
- `assert_focused!(world, id)` — asserts `FocusedMarker` present.
- `assert_on_workspace!(world, window_id, workspace_id)` / `assert_not_on_workspace!(...)` — via `LayoutStrip::index_of`.

Import them via `use crate::{assert_window_at, assert_window_size, assert_focused};` as in `src/tests/tiling.rs:4` and `src/tests/interaction.rs:17`.

## Coverage

**Requirements:** None enforced — no coverage config, no CI coverage step, no minimum threshold. Test count is substantial (~145 `#[test]` functions) concentrated on layout/tiling/state logic.

**View Coverage:** Not configured. If needed, standard tooling applies:
```bash
cargo llvm-cov    # would need installing; not part of the project's tooling
```

## Test Types

**Unit Tests:**
- Inline `#[cfg(test)] mod tests` beside the code. Scope: pure functions (layout math: `clamp_origin_to_viewport`), `LayoutStrip` data structure operations, config parsing (`src/config.rs`), command parsing (`src/commands.rs`, `src/commands/query.rs`), key input parsing (`src/platform/input.rs`).

**Integration Tests:**
- The `src/tests/` tree. Scope: full ECS world driven by `Event` sequences — window spawn/focus/resize/move, workspace switching, multi-display, native tabs, session restore, scroll/swipe physics. These use `TestHarness` + `MockState`.
- `src/tests/state.rs` also shows a `SystemState` pattern for testing extraction logic directly (`src/tests/state.rs:22-58`):
  ```rust
  type QueryStateExtractionState<'w, 's> = SystemState<(
      Query<'w, 's, (&'static ChildOf, &'static LayoutStrip, Has<ActiveWorkspaceMarker>, ...)>,
      // ...
      Res<'w, WindowManager>,
  )>;

  fn extract_query_state(world: &mut World) -> crate::errors::Result<PaneruQueryState> {
      let mut system_state: QueryStateExtractionState<'_, '_> = SystemState::new(world);
      let (workspaces, displays, windows, apps, window_manager) = system_state.get(world);
      PaneruQueryState::extract(&workspaces, &displays, &windows, &apps, &window_manager)
  }
  ```
  Use this when testing a function that takes system params without running the whole schedule.

**E2E Tests:**
- Not used — there is no separate `tests/` directory and no automated GUI testing; manual verification on a real desktop is implied by the domain.

## Common Patterns

**Async Testing:**
- No `async` tests. Bevy's task pools are reduced to one thread in tests (`TaskPoolBuilder::new().num_threads(1)` in `src/tests/harness.rs:214-218`); async tasks spawned by systems (e.g. `BruteforceWindows`) are polled via normal `app.update()` ticks.
- Timers are deterministic through `TimeUpdateStrategy::ManualDuration(100ms)` — advance time by running more iterations, not by sleeping.

**Error Testing:**
- Errors from the OS layer are surfaced by having mock factories return `Err(Error::InvalidWindow)` etc. (e.g. `expect_pid().returning(... ok_or(Error::InvalidWindow))`, `src/tests/mocks.rs:351-358`), then asserting the system's degraded behavior (window skipped, marker removed) rather than a panic.
- Serialization failure cases use `.expect("Failed to serialize")` on the happy path and assert equality after round-trip (`src/tests/state.rs:99-102`).

**Conventions specific to this suite:**
- `unwrap()`/`expect()` are acceptable and common in test code (setup invariants, query `.expect("layout strip")`); the no-`unwrap` rule applies to production systems only.
- Long tests carry `#[allow(clippy::too_many_lines)]` (`src/tests/tiling.rs:10`).
- Unused-but-useful harness helpers carry `#[allow(unused)]` (`src/tests/harness.rs:125,141`) — keep the full harness API available even when only some tests use it.
- Test names describe behavior: `test_window_shuffle`, `modifier_scroll_uses_native_momentum_without_synthetic_velocity`, `native_fullscreen_transition_removes_window_from_original_strip_without_focus_marker` (`src/tests/interaction.rs:22,40`).

---

*Testing analysis: 2026-07-25*
