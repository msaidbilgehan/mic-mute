---
paths:
  - "src/**/*.rs"
---

# Rust Rules (mic-mute)

Code-level conventions for `src/**/*.rs`. Threading, lock order, the mute model, and settings compatibility live in `CLAUDE.md`.

## Errors

- Return `anyhow::Result`. Add `.context("Failed to …")` where a module hands errors upward, as `ui.rs` and `shortcuts.rs` do.
- In `mic.rs`, build errors with `anyhow!` in lowercase with no trailing period, naming the operation and the `AudioDeviceID`; route every OSStatus through `status_result` (`system_status_result` for the system object, which holds the device list).
- Never add `unwrap`, `expect`, `panic!`, or panicking indexing on paths reachable from the event-loop closure or the signal watcher — a panic exits without restoring muted devices. Log with `log::error!` and continue.
- Keep deliberate fallbacks (`unwrap_or(false)` on readback, `.ok()`) local; never discard an error that changes what the user hears or sees.

## Logging

- Use the `log` macros; never `println!` or `eprintln!`.
- Every line also reaches the unified log as public text (`logging.rs`), which `log show` reads long after the app has quit: never log secrets, tokens, or a URL's query or fragment.
- Per-tick and per-device detail is `trace!` (the poll runs it up to five times a second); `info!` is for lifecycle events; `error!` is for failed operations.
- A failure that can repeat on every poll goes through `mic::FailureLog`, never a bare `error!`; log it once where it is handled, not at every layer it passes.
- Refer to devices as `audio device {id}: {name}`, with positional `{}` arguments like the surrounding code.

## CoreAudio FFI (`mic.rs`)

- Call `objc2_core_audio::AudioObject*` with `NonNull` arguments on the Input scope and Main element; never use per-channel elements.
- Check `AudioObjectIsPropertySettable` before every write. Return `Ok(None)` when the property is missing (`kAudioHardwareUnknownPropertyError`, `'who?'`) or not settable, and `Err` for any other non-zero OSStatus.
- Size variable-length properties with `AudioObjectGetPropertyDataSize`, allocate through `AudioBufferListAllocation`, and validate the returned size before slicing; `has_input_channels` is the template. Never read into a fixed-size stack `AudioBufferList` (a stack overflow fixed in `8982124`).
- Confirm every write by reading it back on the control it wrote (`wait_for_native_mute`, `is_volume_muted`); never report success from the write call alone.
- Add each new device operation to the `AudioBackend` trait and implement it in both `CoreAudioBackend` and the test `FakeBackend`.
- Declare four-char selectors as hex constants with the code in a comment (`0x766d7663; // 'vmvc'`).
- Listen with `AudioObjectAddPropertyListener` and `device_changes::property_listener`, never the block API, with a `&'static DeviceChangeSignal` as client data: a listener can run while it is being removed, so its client data must never be freed. Register only properties the object has (`AudioObjectHasProperty`), remove exactly what was registered, and keep the listener body to atomics and posting to the loop. Registration goes through `MicController` (`start_listening`, `sync_listeners`, `stop_listening`), which removes every listener before the exit restore.

## Objective-C and AppKit (objc2)

- Call AppKit and Foundation through objc2-app-kit's and objc2-foundation's typed methods (`popup.rs`, `popup_content.rs`, `about.rs`, `feedback.rs`), which fix every argument's type; debug builds also check each message's type encoding and panic on a mismatch. A method missing from the bindings is usually behind a crate feature: enable it in `Cargo.toml`, keeping the lists sorted (`NSView::layer` needs `objc2-quartz-core`).
- Use `objc2::msg_send!` only where a binding is missing or would panic, with exact types: `NSInteger`, `NSUInteger`, `CGFloat`, `bool`, and `Option<&AnyObject>` for a nullable object. The bindings panic when a method declared nonnull returns nil: fetch such objects with `objc2::msg_send!` into an `Option<Retained<_>>`, as `launch_at_login.rs::main_app_service` does.
- Hold objects in `Retained<T>`, which releases once on drop; never send `retain` or `release`. A superview, stack view, or alert keeps what it holds alive; `PopupContent`'s references only keep its views for `render`. tao's `ns_window` and `ns_view` pointers are borrowed for the window's lifetime (`popup.rs::ns_window`); `MonitorHandle::ns_screen` hands over a reference, which `Retained::from_raw` takes (`screen_menu_bar_height`).
- Views, windows, `NSAlert`, `NSScreen`, and `NSApplication` are `MainThreadOnly`: create them with a `MainThreadMarker`. `MainThreadMarker::new()` is `None` off the main thread: return an error then, never panic (`Popup::new`). A function that reaches them without holding one takes the marker as a parameter (`popup.rs::position_on`).
- Take AppKit enum values from objc2-app-kit's typed constants (`NSTextAlignment::Center`, `NSStackViewGravity::Center`), never literals: some differ by architecture, and the bindings follow the SDK headers. `NSTextAlignment` uses the iOS values on arm64 (`TARGET_ABI_USES_IOS_VALUES`: Center = 1, Right = 2) and the macOS values on x86_64 (Center = 2, Right = 1); `popup_content.rs` tests the constants it uses.
- Declare an Objective-C class (`define_class!`) only to receive AppKit callbacks such as control actions, as `about.rs` does (`AboutButtonTarget`): leave it unnamed (see the next bullet), give each method its exact selector and argument types, and keep the body to recording the event for the loop closure in an atomic ivar — no AppKit calls, no locks, no panics (a panic can't unwind into Objective-C and aborts). Controls don't retain their target (`setTarget` is `unsafe` for that reason): hold its `Retained` for as long as any control points at it.
- Notification observers are classes too, defined with objc2's `define_class!` (`session_events.rs`): one method per notification, taking an `Option<&NSNotification>`, each only setting a bit in an atomic the loop takes, so a delivery on another thread still counts. Leave such a class unnamed: objc2 then names it after its module, and a `#[name]` another class already has panics. Add the observer to the distributed center with `NSNotificationSuspensionBehavior::DeliverImmediately`, so a suspended center never holds an event back, and remove it from every center before releasing it.
- Off the main thread, touch only CoreAudio, CoreMediaIO, and AVFoundation queries, and wrap Cocoa calls in `objc2::rc::autoreleasepool` (see the camera thread in `event_loop.rs`). Never call AppKit there; objc2-app-kit enforces it where a binding takes a `MainThreadMarker` (`set_dock_visible`).
- `camera.rs` reaches AVFoundation through objc2-av-foundation, which links the framework, so it needs no `#[link]` block. It fetches `devicesWithMediaType:` with `msg_send!` into an `Option`: the binding is deprecated and declares the array nonnull.
- Reach `UNUserNotificationCenter` only through `notification::notification_center`, which returns `None` outside an app bundle: there `currentNotificationCenter` raises an Objective-C exception that aborts the process. Its completion blocks (`block2::RcBlock`) run on its own queues; like `define_class!` methods, they only store into an atomic (`SharedPermission`).
- `NSSound::soundNamed` returns a sound AppKit keeps under its name (`feedback.rs`); the `Retained` is the caller's own reference, so dropping it leaves AppKit's.

## objc2 windows (`settings_window.rs`)

- A window kept in `Retained` needs `setReleasedWhenClosed(false)`: its close button then only hides it, and the app reuses it on the next `show`. Detect a close by reading `isVisible` on the next loop pass rather than through a delegate.
- Lay out with `NSGridView` and `NSStackView` and size the window to its content's `fittingSize` (`fit_to_content`, which keeps the top edge); give every wrapping label a `preferredMaxLayoutWidth`.
- Every control sends one action to one `SettingsTarget`, which records the sender's `tag` as a bit in its atomic ivar; the loop takes the bits (`take_changed`) and reads each changed control's value itself.
- Refresh controls from a pure view model (`View`) and touch only the fields that differ: a programmatic `setState:` or `selectItemAtIndex:` sends no action, so a refresh never loops back as a change.
- The shortcut recorder's key monitor (`NSEvent::addLocalMonitorForEventsMatchingMask_handler`, a `block2::RcBlock`) runs on the main thread inside AppKit's event dispatch. It reads only the event, `keyCode` and `isARepeat` only on key-down events (they raise an exception on other types), and leaves the rest to `KeyCatcher`'s `Cell`s, which the loop takes; it returns null to swallow an event. Remove it with `NSEvent::removeMonitor` before its window goes (`Built`'s `Drop`).
- A key code is a physical key: map it through `KEY_CODES`, the inverse of global-hotkey's own table, never through the typed characters, which change with the keyboard layout.

## Tests

- Keep tests in a `#[cfg(test)] mod tests` at the end of the module, named for the behavior (`restore_on_exit_leaves_preexisting_native_mute_unchanged`).
- Mute logic: build controllers with `MicController::with_backend(FakeBackend …)`, `Device::native(…)`-style fixtures, and `store::MemoryStore` journals; extend `FakeBackend` when a scenario is missing. Never call `MicController::new()` or `mic::restore_from_journal_file()` from tests: they read the real `restore.json` and may change devices.
- Time-dependent logic takes `now: Instant` (`InputLevels::save_if_due`, `MicController::mute_all_at`, `WriteRetry::is_due`): production passes `Instant::now()`, tests pass synthetic instants. Tests never sleep; `FakeBackend::readback_retry_delay` is zero.
- Settings: exercise serde with `serde_json::from_str` / `to_string` on literals, and file behavior with `Settings::load_from` / `save_to` on a unique temp dir. Never call `Settings::save`, `load`, or `mtime` — they use the real `~/Library/Application Support/mic-mute/settings.json`.
- Never call `launch_at_login::{reconcile, set, set_dock_visible}`, `Shortcuts::new` / `reload` (they register a real global hotkey), `UI::new`, `Tray::new`, `Popup::new`, `AboutWindow::show`, or `SettingsWindow::show` from tests, nor `about::run_action` with Open settings.json or Reset Settings, or `about::open_settings_file`: they write the real settings file, and Open settings.json opens it.
- Never call `SMAppService` from tests (`mainAppService`, `registerAndReturnError`, `unregisterAndReturnError`, `status`, `openSystemSettingsLoginItems`): they read or change the real login item. Test the planners instead (`login_item_action`, `plist_plan`, `login_item_state`, `mechanism_for`).
- Never call `utils::open_sound_input_settings` from tests: it opens System Settings. Check `SOUND_INPUT_SETTINGS_URLS` instead, as `utils.rs` does.
- Never play a sound or post a notification from tests: don't call `feedback::play` or `preload_sounds`, `Feedback::new`, `set_options`, or `on_mic_update` with either option on, or `Notifier::set_enabled(true)` or `post`. Test `change_to_report`, `Sound::name`, and `is_app_bundle` instead.
- Extract pure helpers from FFI-heavy code and test those (`parse_shortcut`, `should_show_popup`, `monitor_contains_physical_position`, `popup_origin`).
- Test a callback class's method by messaging an instance directly with `msg_send!`, with a fake sender class (`define_class!` in the test module) if it reads one (`about.rs`); never through a window. Test threads aren't the main thread and have no `MainThreadMarker`, so keep callback classes `AnyThread` (the default for an `NSObject` subclass).
- Test the shortcut recorder through its pure parts (`record_press`, `advance`, `catch`, `KeyCatcher`) with synthetic key codes and modifier flags; never create its window or monitor, or pause a real hotkey (`Shortcuts::suspend` / `resume`).
- Never call `SessionEvents::new` from tests: it registers with the system's notification centers. Message the observer instead, and keep everything that records events in one test: they share one atomic (`session_events.rs`).
- Never register real CoreAudio listeners from tests: no `MicController::start_listening` or listener operation on a `CoreAudioBackend`. Listen through `FakeBackend`, which reports changes as CoreAudio does, with a leaked signal per test (`counting_signal`); call `device_changes::property_listener` directly to test the proc.

## Style

- rustfmt defaults (100 columns, matching `.editorconfig`); there is no `rustfmt.toml` or `clippy.toml`.
- Fix clippy findings rather than adding `#[allow]`; when an allow is unavoidable, add a one-line reason.
- Keep the flat `src/` layout: one module per concern, declared in `main.rs`.
- Name booleans for their meaning; don't spread the legacy `CameraController::muted == !in_use` inversion to new code.
