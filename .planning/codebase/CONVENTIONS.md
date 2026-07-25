# Coding Conventions

**Analysis Date:** 2026-07-25

Project-specific guidance also lives in `AGENTS.md` — it documents the same conventions (Clippy, fmt, tracing, `Result`/`Error`, no-`unwrap()` in systems) and is consistent with the code.

## Naming Patterns

**Files:**
- `snake_case.rs`, using Rust 2018 module paths: a parent file `src/ecs.rs` plus a sibling directory `src/ecs/` holding submodules (`src/ecs/layout.rs`, `src/ecs/params.rs`). No `mod.rs` files.

**Functions:**
- `snake_case`. Systems are plain functions: `pub(crate) fn add_existing_process(...)` in `src/ecs/systems.rs:95`. Triggers/observers end in `_trigger`: `mission_control_trigger` (`src/ecs/triggers.rs:308`), `spawn_window_trigger`. Fallible pure helpers return `Result`: `clamp_origin_to_viewport` (`src/ecs/layout.rs`).

**Types:**
- `CamelCase`. Conventions by role:
  - Marker components end in `Marker`: `FocusedMarker`, `ActiveWorkspaceMarker`, `ActiveDisplayMarker`, `FreshMarker`, `ExistingMarker`, `ReshuffleAroundMarker`, `EnsureVisibleMarker` (`src/ecs.rs:188-232`). State enums like `Unmanaged { Floating, Minimized, Hidden }` (`src/ecs.rs:285`) act as marker-ish components.
  - OS-abstraction traits end in `Api`: `WindowManagerApi` (`src/manager.rs:81`), `WindowApi` (`src/manager/windows.rs`), `ApplicationApi` (`src/manager/app.rs`), `ProcessApi` (`src/manager/process.rs`). Each is annotated `#[automock]` (mockall).
  - Real macOS implementations are suffixed `OS`: `WindowManagerOS` (`src/manager.rs:196`), `WindowOS`, `ApplicationOS`.
  - Bevy `SystemParam` structs get plain names: `Windows`, `ActiveDisplay`, `ActiveDisplayMut`, `GlobalState` (`src/ecs/params.rs`).
  - Resources and newtype components are tuple structs with `#[derive(Deref, DerefMut)]` via `derive_more`: `Position(pub Origin)`, `Bounds(pub Size)`, `WidthRatio(pub f64)` (`src/ecs.rs:255-265`), `SkipReshuffle(pub bool)`, `FocusFollowsMouse(pub Option<WinID>)` (`src/ecs.rs:416-429`).

**Variables:**
- `snake_case`. Test constants are `SCREAMING_SNAKE_CASE` with `TEST_`/`EXT_` prefixes: `TEST_DISPLAY_ID`, `TEST_WORKSPACE_ID`, `EXT_DISPLAY_ID` (`src/tests.rs:13-26`).

## Code Style

**Formatting:**
- `cargo fmt` with default settings — there is no `rustfmt.toml`/`.rustfmt.toml`. Run before finalizing any change (per `AGENTS.md`).
- Edition 2024, `rust-version = "1.89.0"` (`Cargo.toml:4-5`), stable toolchain pinned by `rust-toolchain.toml`.

**Linting:**
- Clippy **pedantic** enabled workspace-wide via `Cargo.toml:14-15`:
  ```toml
  [lints.clippy]
  pedantic = "warn"
  ```
- Run `cargo clippy` before finalizing changes (per `AGENTS.md`). One crate-level allow: `#![allow(clippy::cast_possible_truncation)]` in `src/main.rs:1`.
- Targeted `#[allow(...)]` attributes are used liberally at the function/struct level rather than weakening global lints. By frequency:
  - `#[allow(clippy::needless_pass_by_value)]` (~110 uses) — **required idiom on every Bevy system/trigger**, because systems take `Res`, `Query`, `Commands` etc. by value. Example: `src/ecs/systems.rs:93-95`.
  - `#[allow(clippy::type_complexity)]` (~20 uses) — on complex query tuples and `SystemParam` structs, e.g. `Windows` in `src/ecs/params.rs:201`.
  - `#[allow(clippy::too_many_lines)]`, `#[allow(clippy::too_many_arguments)]`, `#[allow(clippy::float_cmp)]`, `#[allow(clippy::cast_precision_loss)]`, `#[allow(clippy::struct_excessive_bools)]` — sparingly, where refactoring would hurt clarity.

## Import Organization

**Order (observed throughout):**
1. `std` imports
2. External crates: `bevy::...`, `objc2*::...`, `tracing::...`, `derive_more::...`
3. Crate-local: `use crate::{...}` then `use super::{...}`

Example from `src/manager.rs:1-46` and `src/ecs/params.rs:1-24`. Production modules import explicit Bevy paths (`bevy::ecs::system::{Query, Res, SystemParam}`); test files use `bevy::prelude::*`.

**Path Aliases:**
- None (no `use as` aliases). Re-exports are centralized in parent modules instead: `src/manager.rs:33-46` does `pub use app::{Application, ApplicationApi}; pub use windows::{Window, WindowApi, WindowOS, ...};` so consumers write `use crate::manager::{Window, Application}`.
- Test-only re-exports are gated: `#[cfg(test)] pub use process::MockProcessApi;` (`src/manager.rs:48-51`).

## Error Handling

**Core types** — `src/errors.rs`:
- `pub type Result<T> = std::result::Result<T, Error>;` (`src/errors.rs:5`). Always import and use this alias, never `std::result::Result` with explicit error.
- `pub enum Error` (`src/errors.rs:9-26`) with 8 variants: `InvalidWindow`, `InvalidConfig(String)`, `ConfigurationWatcher(String)`, `NotFound(String)`, `PermissionDenied(String)`, `InvalidInput(String)`, `IO(String)`, `Generic(String)`.
- Blanket conversion `impl<T: std::error::Error + Display + 'static> From<T> for Error` (`src/errors.rs:61-71`) maps `std::io::Error` → `IO`, `notify::Error` → `ConfigurationWatcher`, everything else → `Generic` — so `?` works on most fallible calls.
- Constructor `Error::invalid_window(message)` (`src/errors.rs:38-41`) logs the message at `debug!` level before returning the variant — use it instead of the raw variant when context exists.

**Patterns:**
- **Systems do not return `Result`.** Inside Bevy systems/triggers, handle errors with `let-else` + log + early return, or `.inspect_err()`:
  ```rust
  // src/ecs/systems.rs:101-104
  let Ok(app) = window_manager.new_application(&*process.0) else {
      error!("creating aplication from process '{}'", process.name());
      return;
  };

  // src/ecs/systems.rs:141-143
  if let Ok((found_windows, offscreen)) = window_manager
      .find_existing_application_windows(&mut app, &spaces, &config)
      .inspect_err(|err| warn!("{err}"))
  { ... }
  ```
  `.inspect_err(|err| warn!/error!/debug!(...))` is the dominant idiom (~30 uses: `src/ecs/triggers.rs`, `src/ecs/systems.rs`, `src/reader.rs`, `src/manager.rs`).
- **Avoid `unwrap()` in systems** (per `AGENTS.md`). Use `Option` combinators (`map_or_else`, `is_some_and`, `then_some`), `let-else`, or log-and-skip. `.expect()` is reserved for infallible-by-construction setup (e.g. `ctrlc::set_handler(...).expect("setting Ctrl-C handler should succeed")` in `src/main.rs:153`) and for tests, where `unwrap()`/`expect()` are fine.
- **Use fallible command variants:** `commands.get_entity(entity)` + `entity_commands.try_remove::<T>()` / `try_despawn()` instead of panicking forms — see `src/ecs/systems.rs:106-108,148-150`.
- Query errors: `.single().ok()`, `.get(entity).inspect_err(...).ok()` (`src/ecs/params.rs:242-247, 277-279`).

## Logging

**Framework:** `tracing` + `tracing-subscriber` (with `env-filter` feature).

**Subscriber setup:** compact fmt layer to stderr with level/line/file/target, `EnvFilter` defaulting to `"info"` — `src/main.rs:126-138`. The same setup is mirrored for tests behind a `OnceLock` in `src/tests/harness.rs:198-220`. Control verbosity with `RUST_LOG` (e.g. `RUST_LOG=paneru=debug`).

**Patterns:**
- **`#[instrument]` on every system and trigger.** Standard form:
  ```rust
  #[instrument(level = Level::DEBUG, skip_all)]
  ```
  Add selective fields for context: `fields(trigger)`, `fields(display_id)`, `fields(window)` — see `src/ecs/triggers.rs:308`, `src/ecs/display.rs:208`, `src/manager.rs:401`.
- **Level selection:**
  - `DEBUG` — lifecycle events, triggers, window/workspace operations (`src/ecs/workspace.rs`, `src/ecs/focus.rs`, `src/commands.rs`).
  - `TRACE` — hot paths that run every frame or per scroll tick: scroll physics (`src/ecs/scroll.rs`), animation/position commit (`src/ecs/systems.rs:496-967`), layout internals (`src/ecs/layout.rs:659-727`), and most `WindowManagerApi` methods (`src/manager.rs:327,378`, `src/manager/windows.rs:160-500`).
- Manager-level methods add `ret` (log return value) and `skip(self)`: `#[instrument(level = Level::TRACE, skip(self), ret)]` (`src/manager.rs:327`); `skip_all` for noisy params (`src/manager/display.rs:143`).
- Macros used directly: `error!`, `warn!`, `info!`, `debug!`, `trace!` — imported per-file from `tracing`, never via a wrapper. `info!` is reserved for high-signal events (e.g. initialization summary, `src/ecs/systems.rs:204`).

## Comments

**When to Comment:**
- Explain **macOS quirks and non-obvious invariants**, not mechanics. Examples: the overlay run-condition rationale in `src/ecs.rs:72-76`, "bevy's `TerminalCtrlCHandlerPlugin` was not fast enough..." in `src/main.rs:149`, "NSScreen::screen needs to run in the main thread" in `src/ecs/display.rs:384`.

**Doc comments:**
- `///` doc comments on essentially all public items — systems, triggers, SystemParam accessors, trait methods (`src/manager.rs:82-130` is the canonical example).
- Trait methods and systems use `# Arguments` / `# Returns` sections (`src/errors.rs:29-37`, `src/ecs/systems.rs:85-92`). Match this style for new public APIs.

## Function Design

**Size:** Large systems are tolerated when linear (with `#[allow(clippy::too_many_lines)]`), but complex logic is decomposed: layout math is extracted into pure free functions in `src/ecs/layout.rs` (e.g. `clamp_origin_to_viewport`, `strip_has_changed_window`) so it can be unit-tested without a `World`.

**Parameters:**
- Systems take Bevy params by value (`Res<T>`, `ResMut<T>`, `Query<...>`, `Commands`, `Populated<...>`) and always carry `#[allow(clippy::needless_pass_by_value)]`.
- **Group related queries into a `#[derive(SystemParam)]` struct** rather than long parameter lists — the established pattern in `src/ecs/params.rs`:
  ```rust
  #[derive(SystemParam)]
  pub struct ActiveDisplay<'w, 's> {
      strip: Single<'w, 's, (&'static LayoutStrip, Entity, ...), With<ActiveWorkspaceMarker>>,
      display: Single<'w, 's, (&'static Display, Entity, Option<&'static DockPosition>), With<ActiveDisplayMarker>>,
      other_displays: Query<'w, 's, &'static Display, Without<ActiveDisplayMarker>>,
  }
  ```
  Expose intent-named accessors (`display()`, `id()`, `actual_bounds(&Config)`, `other()`), keep fields private. Immutable/mutable variants are separate structs (`ActiveDisplay` / `ActiveDisplayMut`).

**Return Values:** `Result<T>` (the crate alias) for fallible non-system functions; `Option` for lookups (`Windows::find`, `Windows::focused`); systems return `()`.

## Module Design

**Exports:** Parent modules re-export their children's public API (`src/manager.rs:33-46`, `src/ecs.rs` `pub mod` list at lines 39-49). Consumers import from the parent, not deep paths — except `LayoutStrip` which is imported from `crate::ecs::layout::LayoutStrip`.

**Visibility:** `pub(crate)` is the default for systems, triggers, and test support (`src/ecs/systems.rs:95`, `src/tests/harness.rs:30`); `pub` only for cross-module API; `pub(super)` inside `src/ecs/triggers.rs` for trigger fns. `src/ecs/restore.rs` is `pub(crate) mod`.

**Barrel Files:** Not used as a pattern — `src/tests.rs` is the closest thing (module declarations + shared constants + `pub(crate) use harness::*; pub(crate) use mocks::*;`).

## ECS-Specific Idioms (from `AGENTS.md`, verified in code)

- **Marker components for filtering/state:** attach/remove unit markers (`FocusedMarker`, `FreshMarker`, `Unmanaged`) and filter with `With<T>`/`Without<T>`/`Has<T>` — never boolean fields on shared components. Markers live in `src/ecs.rs:188-299`.
- **Change detection to gate expensive OS calls:** `Changed<Position>`/`Changed<Bounds>`/`Changed<LayoutStrip>` drive window repositioning, resizing, and overlay refresh — `src/ecs/systems.rs:933,970`, `src/ecs/layout.rs:875-1295`, and the `overlay_dirty` run condition in `src/ecs.rs:77-86`. Use `Changed<T>` (plus `Added<T>`/`RemovedComponents<T>`) rather than unconditionally calling macOS APIs every frame.
- **Triggers & observers for reactive logic:** define a trigger type (e.g. `SpawnWindowTrigger(pub Vec<Window>)`, `WMEventTrigger`), fire with `commands.trigger(...)` / `world.trigger(...)`, register observers in `register_triggers` via `app.add_observer(...)` (`src/ecs.rs:163-186`). Message-driven triggers use `MessageReader<Event>` (`src/ecs/triggers.rs:308-321`).
- **System grouping:** all systems are registered in `register_systems` (`src/ecs.rs:59-160`) across `Startup`, `PreUpdate`, `Update`, `PostUpdate` with `.chain()` for ordering and run conditions (`run_if(resource_exists::<Initializing>)`, `on_timer(...)`, custom closures like `not_swiping`). Add new systems there, in the correct schedule.
- **Main-thread / NonSend:** AppKit/CoreGraphics-touching objects that aren't thread-safe are accessed via `Option<NonSend<T>>`/`Option<NonSendMut<T>>` — `PlatformCallbacks` (`src/ecs/systems.rs:589`, `src/ecs/display.rs:375`), `OverlayManager`, `MenuBarManager` (`src/menubar.rs:307`), the config `Watcher` (`src/ecs/triggers.rs:1123`). Note `WindowManager` is an exception: it's a plain `Resource` wrapping `Box<dyn WindowManagerApi>` because the trait requires `Send + Sync` (`src/manager.rs:80-81,190-192`) — which is also what makes it mockable in tests.
- **FFI boundaries:** ECS systems never call `objc2`/SkyLight directly; they go through `WindowManager`/`WindowApi`/`ApplicationApi` in `src/manager/` and `src/platform/`.

---

*Convention analysis: 2026-07-25*
