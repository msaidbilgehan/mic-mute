# TODOs

Planned work for this fork. Line numbers refer to `main` @ `9d9d6d5`; findings were verified on 2026-10-15.
Tick boxes in the same commit as the change, and keep `CLAUDE.md` consistent when an item changes a rule or command.

Priorities: **P0** blocks daily use (installing, or leaving mics in the wrong state) · **P1** important · **P2** enhancement.

Suggested order: #3 → #4 (one-command install), then #1 + #2 (they share the restore journal), then #6; reinstall with `task install` after each. Since #1 (2026-10-16), every quit path restores mics; after a crash, `kill -9`, or power loss, the next launch restores them from `restore.json`.

## P0 — Before daily use

### 1. Mics stay muted after quitting via Dock, ⌘Q, logout, AppleScript, a panic, or a crash

**Problem.** Only tray → Exit and SIGINT/SIGTERM restore device state.

- Every `NSApp terminate:` path (Dock → Quit with Show in Dock on, ⌘Q, logout, `osascript -e 'quit app "Mic Mute"'`) reaches the closure as `Event::LoopDestroyed` (tao `app_delegate.rs:131` → `app_state.rs:272-274`), which falls into `_ => {}` (`event_loop.rs:146`).
- Panics skip restore: there is no panic hook, and the closure unwraps at `event_loop.rs:60-61,66,136,143,247`.
- SIGKILL, crashes, and power loss can't run code. On the next launch, leftover mutes are adopted as user intent (`mic.rs:425-426`) and their saved volumes are lost.

**Fix plan.**

- [x] Handle `Event::LoopDestroyed` in `event_loop::start` by calling `restore_microphone_on_exit`; this covers every `NSApp terminate:` path.
- [x] Persist a restore journal at `~/Library/Application Support/mic-mute/restore.json`. Key it by device UID (`kAudioDevicePropertyDeviceUID`, available in objc2-core-audio), because `AudioDeviceID`s change across reboots and re-plugging. Store whether the app set native mute and the pre-mute volume. Write it atomically (temp file + rename) before changing a device, and only when its content changes — not on every 200 ms poll. Delete it after a clean restore.
- [x] On startup, if the journal exists (unclean exit), restore those devices before `MicController::new()` so they aren't adopted as user intent.
- [x] Install a panic hook (`std::panic::set_hook`) that restores from the journal without taking the controller lock (the panicking thread may hold it), then calls the previous hook.
- [x] Add SIGHUP and SIGQUIT to the shutdown flag (`main.rs:62-69`).
- [x] Replace the `unwrap`s in the event-loop closure with `log::error!`.
- [x] Add `device_uid()` to `AudioBackend`; implement it in `CoreAudioBackend` and `FakeBackend`.

**Verify.** FakeBackend tests: journal written on mute and cleared on restore; a stale journal is restored at startup and not adopted; a failed restore keeps its entries. Manual: Dock → Quit (Show in Dock on), ⌘Q, `osascript -e 'quit app "Mic Mute"'`, log out and in, `kill -9` then relaunch.

Verified 2026-10-16 on the installed app without changing any device: launched with a `restore.json` that listed a connected device and a missing one. Startup restored the connected device and dropped it from the journal. AppleScript quit and SIGTERM, SIGINT, SIGHUP, and SIGQUIT each exited with status 0 after the exit restore, which kept the missing device's entry. After `kill -9`, the next launch restored from the journal. Not yet run: Dock → Quit, ⌘Q, logout, and a mute–quit cycle on a real microphone (needs a mic that no other mute app holds).

### 2. Unmute can jump input volume to maximum (1.0)

**Problem.** Devices without native mute are muted by setting input volume to 0. On unmute, `mute_via_volume` restores `saved_volumes[id]` — or `1.0` when nothing was saved — and removes the saved value before the write succeeds (`mic.rs:604-608`). A failed write loses the original level for good.

**Requirement.** Unmute restores the exact pre-mute level.

**Fix plan.**

- [x] Keep the saved volume until the write is confirmed by readback; on failure keep it for the next attempt and for `restore_on_exit`.
- [x] Make an unknown level practically impossible:
  - Key saved levels by device UID and persist them in the restore journal (item 1), so restarts, crashes, and re-plugging (new `AudioDeviceID`, same UID) keep them.
  - Record each volume-controlled device's last audible level whenever Mic Mute sees it audible — at startup, on hot-plug, and while unmuted (each poll, or item 13's property listeners) — and persist it per UID, writing only when it changes.
  - On unmute, restore in this order: the level saved when the app muted the device → the last audible level ever recorded → `UNMUTE_FALLBACK_VOLUME`.
- [x] `UNMUTE_FALLBACK_VOLUME = 0.5` (decided 2026-10-15). It applies only to a device Mic Mute has never seen audible (e.g. first seen already at 0); log a warning when it's used.
- [x] Tests: change `fallback_unmute_uses_audible_default_when_saved_volume_was_zero` (`mic.rs:1070-1081`, asserts 1.0 today) to expect 0.5; add tests for a failed unmute write keeping the saved level, a re-plugged device (new ID, same UID) restoring its level, and a restart restoring from the persisted record.

### 3. Build tooling is broken (`task build`, `task release`, `task debug`)

**Problem.**

- `task build` and `task release` run an unpinned `cargo install cargo-bundle` (`Taskfile.dist.yaml:70`), which installs 0.12.0 (2026-09-20). 0.12 rejects unknown keys (`#[serde(deny_unknown_fields)]`) and moved macOS settings into a nested `[package.metadata.bundle.osx]` table, so the flat `osx_url_schemes` / `osx_minimum_system_version` keys (`Cargo.toml:66-67`) fail at `cargo bundle`.
- `task debug` runs `./target/debug/mvcm` (`Taskfile.dist.yaml:38`); the binary is `mic-mute`, and nothing builds it first.
- The release `status:` checks look for `apple-codesign` / `toml-cli` (`:98-99`), but those crates install `rcodesign` / `toml`, so both reinstall on every release.
- `osx_minimum_system_version = "10"`, but the binary requires macOS 11.0.
- `task test` runs `cargo test` without `--locked` (`:57`); CI uses `--locked`.

**Fix plan.**

- [x] Migrate the bundle metadata to the cargo-bundle 0.12 schema:

  ```toml
  [package.metadata.bundle.osx]
  minimum_system_version = "11.0"
  url_schemes = ["com.brettinternet.mic-mute"]
  info_plist_exts = ["assets/Info.plist.ext"]
  ```

  `assets/Info.plist.ext` holds the `NSCameraUsageDescription` key/string pair; cargo-bundle 0.12 inserts it verbatim inside `<dict>`, escaping only `&` (`osx_bundle.rs:146-150,299-302`). This replaces the PlistBuddy `|| true` steps (`Taskfile.dist.yaml:78,106`). Fix the wording too: the app only detects camera use; it doesn't control the camera.
- [x] Pin every tool install with `--locked` and an exact version (`cargo-bundle@0.12.0`, `apple-codesign`, `toml-cli`, `watchexec-cli`), and point the `status:` checks at the real binaries (`cargo-bundle`, `rcodesign`, `toml`, `watchexec`).
- [x] `task debug`: `cargo build`, then `./target/debug/mic-mute` — or delete the task.
- [x] `task test`: `cargo test --locked`.
- [x] Add `desc:` to every public task so `task -l` lists them; mark `build-target` `internal: true`.
- [x] `.taskfiles/setup.yaml:16`: replace `mise trust -q -a` with `mise trust -q`. `-a` also trusts mise configs in every parent directory, and trust is path-based, so a later malicious edit there would run without a prompt.

### 4. Build and install pipeline script

**Goal.** One command takes a clean checkout to an installed, running app (replacing the paid mute app). Depends on item 3.

- [x] `scripts/install.sh` (bash, `set -euo pipefail`), exposed as `task install`:
  1. Preflight: macOS ≥ 11 on arm64, Xcode Command Line Tools, rustup target `aarch64-apple-darwin`, pinned cargo-bundle.
  2. Gates: `cargo fmt -- --check`, `cargo clippy --locked --release -- -D warnings`, `cargo test --locked`.
  3. `cargo bundle --release --target aarch64-apple-darwin`.
  4. Ad-hoc sign with `codesign --force --deep --sign - "Mic Mute.app"` and check with `codesign --verify`. Locally built apps carry no quarantine flag, so no Gatekeeper bypass is needed.
  5. Stop a running instance with `pkill -TERM -x mic-mute` (the SIGTERM path restores mics) and wait for it to exit.
  6. Install to `/Applications/Mic Mute.app` (`--user` → `~/Applications`), replacing atomically: copy to a temp path, then `mv`.
  7. If `~/Library/LaunchAgents/com.brettinternet.mic-mute.plist` exists, rewrite its `ProgramArguments` to the installed binary; the app writes that path only when the setting is toggled (`launch_at_login.rs:19`).
  8. Launch with `open -a "Mic Mute"` and confirm the process is running.

  Flags: `--dry-run`, `--skip-checks`, `--no-launch`.
- [x] `task uninstall`: SIGTERM the app, remove the bundle and the LaunchAgent plist; keep settings unless `--purge`.
- [x] Optional: a GitHub Actions release workflow on a macOS runner that builds the DMG, publishes sha256 checksums, and adds build provenance (`actions/attest-build-provenance`), with every action pinned by SHA. Upstream releases are built locally, unnotarized, with no checksums.

### 21. Defects found by the 2026-10-16 audit

A safety audit of the mute paths found ways to leave microphones live, or muted, against the user's intent.

- [x] **"Mute" could unmute the working mics.** `toggle(None)` flipped `desired_muted`, while the tray, popup, and sounds follow `muted`. When one device's mute write kept failing, a mute left the other devices muted and the UI at "Mute"; pressing it again (or the shortcut, or a `toggle` URL) ran `mute_all(false)` and unmuted them. A toggle now flips `muted`, so that press mutes again and retries the failed device, and a toggle while every device reads muted unmutes, as the UI says. Push-to-talk and the URL `mute`/`unmute` actions set their state explicitly; `mute-state.json` still records the intent each request leaves.
- [x] **One failing device stopped startup or blocked every mute.** `main` unwrapped `MicController::new`, and one error from a device's name, input-stream check, or mute read made the launch, `mute_all`, or the poll return before it wrote to any other device. Those errors are now per device: such a device counts as failed for the pass (one combined error through `FailureLog`, Failed in the Microphones menu), is still muted with the others since it may be a microphone, never counts as muted while unreadable, and is read again by the next poll. `MicController::new` can't fail: a device list it can't read leaves it without devices until a poll lists them, so the tray and shortcut keep working. `FakeBackend` fails names (`fail_name`) and the device list (`fail_device_ids`).
- [x] **The startup restore left mics live before the launch mute.** `MicController::new` restored a leftover `restore.json` (an unclean exit), unmuting the journaled devices, and only at `StartCause::Init` did the loop mute again for push-to-talk, `launch_state: "muted"`, or `"last"` after a muted quit: the mics were live for about 0.1–1 s. `main` now loads `mute-state.json` and decides the launch mute before it builds the controller (`LaunchIntent`). A launch that mutes adopts the journal as the app's own mute (its devices stay muted and journaled, and the intent starts muted); only a launch that keeps the hardware state restores. An entry for an absent device still waits for it: it is restored when the device is back while unmuted, and otherwise stays the app's own mute until the next unmute or exit.
- [x] **The signal watcher could exit with the mics muted.** It called `restore_microphone_on_exit`, which released the controller lock, and then `std::process::exit(0)`. In between, the main thread could take the lock for a mute it was waiting to run (the shortcut, the tray, push-to-talk, a session event, or the launch mute) and mute again, leaving the mics muted until the next launch. The watcher now restores and exits under one write guard (`main.rs::restore_then_exit` with `restore_locked_microphone_on_exit`); the main thread then waits on the lock until the process ends.
- [x] A tray toggle, Settings-window change, Microphones click, or About → Reset made within 2 s of a hand edit saved the in-memory settings over it, and after a startup with an invalid file, replaced the fixed file with defaults plus the change: `save_to` only checked validity, and `take_own_save` then skipped the reload. Every change now saves through `Settings::update`, which reloads a file whose mtime or contents differ from `Settings::file_version` and saves the change into it; tray clicks set the opposite of what the menu showed instead of toggling the reloaded value.

## P1 — Important

### 5. Security advisories: RUSTSEC-2026-0204 (crossbeam-epoch) and RUSTSEC-2026-0190 (anyhow)

**Problem.** crossbeam-epoch 0.9.18 has a vulnerability (invalid pointer dereference in the `fmt::Pointer` impl for `Atomic` / `Shared`); anyhow 1.0.102 has an unsound `Error::downcast_mut`. `src/` calls neither API (no `downcast`, crossbeam, or `{:p}` use), so exposure is theoretical. `cargo deny check advisories` also reports five unmaintained crates: async-std, paste, rustybuzz, ttf-parser, proc-macro-error.

**Solution path** (verified in a scratch copy on 2026-10-15: fmt, release clippy, 46/46 tests, release build, and `cargo deny check advisories` all pass):

- Minimal: `cargo update -p anyhow -p crossbeam-epoch` → anyhow 1.0.104, crossbeam-epoch 0.9.21; only those two packages change.
- Full — clears every advisory and drops the compiled crate count from 240 to 142 (lockfile 413 → 295 packages):
  - [x] `image = { version = "0.25", default-features = false, features = ["png"] }` — the app only encodes PNG (`popup_content.rs:85-88`). Drops rayon / crossbeam-epoch, the AVIF encoder (rav1e, paste), and the other formats.
  - [x] `resvg = { version = "0.47", default-features = false }` — the icons contain no text or raster images. Drops rustybuzz, ttf-parser, fontdb (resvg already depends on usvg with `default-features = false`).
  - [x] Remove `async-std` (RUSTSEC-2025-0052). Its only use is the 1 s `HidePopup` delay (`event_loop.rs:64-67`); fold a `hide_popup_at: Option<Instant>` deadline into the existing `WaitUntil` scheduling (a `std::thread` + `EventLoopProxy` also works).
  - [x] `cargo update -p anyhow`.
  - [x] Add `deny.toml` with `[graph] targets = ["aarch64-apple-darwin"]` so Linux-only crates (proc-macro-error via GTK) don't fail the check, and run `cargo deny check advisories` in CI in place of the archived `actions-rs/audit-check@v1` (`audit.yaml`).

### 6. Default keyboard shortcut → ⌘⇧M

**Decision (2026-10-15).** The default becomes ⌘⇧M: `{ "modifiers": ["shift", "meta"], "key": "M" }`. ⌘M was rejected because it is the system-wide Minimize shortcut.

**Trade-off.** A global hotkey (Carbon `RegisterEventHotKey`) takes its combination away from every app while Mic Mute runs. By default ⌘⇧M opens, for example, VS Code's Problems panel and Chrome's profile menu (Microsoft Teams uses it for mute anyway). Today's ⌘⇧A shadows Finder's Go → Applications and Zoom's mute.

- [x] `ShortcutConfig::default()` (`settings.rs:12-19`): `key: "M"`. About → Reset Settings follows automatically (it uses `ShortcutConfig::default()`).
- [x] Update `test_default_shortcut` (`settings.rs`), the default JSON and the "default is" sentence in `docs/settings.md`, and `README.md:11`.
- [x] Settings files that already set `mic_shortcut` keep their value; only setups without one change.

### 7. Invalid `settings.json` silently resets every setting

**Problem.** `Settings::load()` returns full defaults on any parse error and logs nothing (`settings.rs:49-51`). The 2 s reload applies them live: the shortcut resets and the LaunchAgent is deleted (`ui.rs:126`). Startup reconciliation (`main.rs:46-50`) can then overwrite the broken file. Unknown fields are dropped on the next save.

- [x] Return `Result` from loading; on a reload error keep the last good settings and log a warning with serde's line and column.
- [x] At startup with an invalid file, run on in-memory defaults without saving over the file, and tell the user (tray label or notification).
- [x] Save atomically: write `settings.json.tmp`, then rename.
- [x] Record the mtime after the app's own `save()` so its writes don't trigger a full `apply_settings` 2 s later.

### 8. Read-only native mute hides the volume fallback

**Problem.** `is_muted` trusts a readable native mute even when it isn't settable (`mic.rs:462-482`). After the fallback mutes such a device by volume, it still reads "unmuted", so every 200 ms poll errors and `muted` never becomes true. Skipped devices with a readable mute block the muted state the same way.

**Blocks daily use on this Mac.** Microsoft Teams installs a virtual input, "Microsoft Teams Audio" (UID `MSLoopbackDriverDevice_UID`). It reports native mute and input volume as settable and accepts both writes with `noErr`, but its readback never changes (checked 2026-10-16 with a CoreAudio test tool). From `mic.rs`: muting it fails both the native and the volume path, so every mute logs an error, `muted` stays false while the device is connected, and the tray and popup show "unmuted" although the built-in mic is muted.

- [x] Track how each device is controlled (native or volume) and read its state from the same control; treat a read-only native mute as absent.
- [x] Treat a device whose writes are accepted but never confirmed by readback, on both controls, as uncontrollable: leave it out of `is_muted_all` and enforcement, warn once, and probe it again every 60 s and when it reconnects. A native mute that ignores writes falls back to input volume first. Decide from behavior, never from the device name.
- [x] Teach `FakeBackend` a readable-but-unsettable mute and a device that ignores accepted writes; add regression tests.

### 9. Shortcut parsing and reload

**Problem.**

- `Shortcuts::reload` unregisters the old hotkey before registering the new one (`shortcuts.rs:57-67`). A failure leaves no working hotkey, and the `?` in `apply_settings` (`ui.rs:110`) skips the popup, Dock, and login sync.
- Modifiers are case-sensitive and unknown ones are dropped silently; an unparseable key silently becomes `A` (`shortcuts.rs:14-32`). `["Shift", "Cmd"]` registers a bare `A` system-wide.
- Three separate parsers (`shortcuts.rs`, `tray.rs:37-53`, `about.rs:10-23`) disagree.

- [x] One parser returning `Result<HotKey>`, case-insensitive modifiers, errors for unknown tokens; derive the tray label and About text from it.
- [x] Register the new hotkey first; unregister the old one only on success; on failure keep the old one and log.
- [x] Reject shortcuts without modifiers unless the key is F13–F24, so a plain key is never stolen system-wide.
- [x] Don't abort `apply_settings` on a hotkey error.

### 10. Smaller defects

- [x] `popup_content.rs:70`: `NSALIGNMENT_CENTER: i32 = 1` and `GRAVITY_CENTER: i32` (`:200`) were passed as 32-bit values; both are `NSInteger` (`i64`). The value 1 already centered on Apple silicon, where `NSTextAlignment` uses the iOS values (Center = 1, Right = 2); only Intel Macs use Center = 2. The constants are now `i64`, and the alignment is chosen per architecture.
- [x] Create the tray icon and hotkey manager after the event loop starts (`StartCause::Init`), as tray-icon's macOS docs require (`ui.rs:36-46`). The loop now creates the whole `UI` on Init; if that fails, it restores the mics and exits with code 1.
- [x] Default log level `info` in release builds; `main.rs:39` defaults to `trace`, and `task release`'s `RUST_LOG: warn` affects only the build.
- [x] Mute enforcement pauses while the About `NSAlert` runs modally (`about.rs:56`); use a non-modal window. About is now the alert's panel shown without `runModal`; its buttons record the click for the event loop, which closes the window and runs the action.
- [x] About → Open Settings does nothing on a fresh install (the file doesn't exist yet); About → Reset Settings resets only `mic_shortcut` although `event_loop.rs:196` says all settings.
- [x] Refresh the LaunchAgent's `ProgramArguments` at startup when enabled — the path goes stale when the app moves (or do item 15).
- [x] Remove the unused direct dependencies `block` and `core-foundation-sys` (`Cargo.toml:31,33`).
- [x] Remove `unsafe impl Send/Sync` for `UI` and `Tray` (`ui.rs:22-23`, `tray.rs:55-56`) if it compiles; they hide cross-thread AppKit use from the compiler.
- [x] While muted, every 200 ms poll re-rasterized three SVG icons, re-encoded PNGs, and reset the tray icon. Each (icon, color) variant is now rendered once per process, and the tray and popup redraw only what changed.
- [x] While muted, every 200 ms poll wrote mute to every device that was already muted (`mic.rs` `mute_native` called `set_mute` unconditionally) and queried `active_device_name()`, which nothing showed (`event_loop.rs:54`, `popup.rs:166`). A control is now written only when it doesn't already read the target state, and the query is gone.
- [x] A persistent device failure logged `error!` every 200 ms twice: per device in `mute_all` (`mic.rs:652`) and combined in `update_mic`. The per-device line is now `trace!`; the combined one goes through `FailureLog`: `error!` at first, identical repeats at `trace!`, one `error!` a minute with their count, `info!` on recovery.
- [x] About → Open Settings spawns `open` and never waits for it (`about.rs` `open_settings_file`), so every click leaves a zombie process until the app quits. Reap it (e.g. wait on a short-lived thread) or open the file with `NSWorkspace`. It still spawns `open -t`, so the file opens in the default text editor, and a short-lived thread waits for it (`wait_for_open`). `NSWorkspace openURL:`, tried in between, opened it in the app macOS uses for `.json` files, which may be Xcode.
- [x] Pin GitHub Actions by commit SHA in both workflows; replace `actions/checkout@v3` in `audit.yaml`.
- [x] README: three identical CI badges (`README.md:58-60`) and "the the" (`:54`).

## P2 — Features

### 11. Push-to-talk

Hold the shortcut to unmute temporarily, release to mute again. global-hotkey already delivers `HotKeyState::Released`; the loop ignores it (`event_loop.rs:210-219`). Setting: `mode: "toggle" | "push_to_talk"`.

- [x] `mode` setting (default `"toggle"`). The loop handles every queued hotkey event, `Pressed` and `Released`, through the pure state machine `push_to_talk::PushToTalk`: a press unmutes, a release mutes, a repeated press changes nothing, and a release without a press still mutes.
- [x] The resting state is muted: a launch in the mode mutes, and so does switching into it.
- [x] A lost release never leaves a mic open. Every session event that can swallow it mutes in push-to-talk mode: screen lock (`com.apple.screenIsLocked`), screen saver start, system sleep, display sleep, and a switch to another user (`session_events.rs`, an observer class declared with `ClassDecl`). So does a re-registered hotkey while held (the release would carry the old id) and leaving the mode while held.
- [x] The tray's Mute item and the URL actions work as in toggle mode; their unmute lasts until the next release or session event. Documented in `docs/settings.md`.

### 12. Settings window with a shortcut recorder

A native window for the shortcut (recorder), the toggles, and popup options, so nobody edits JSON; it surfaces item 7's validation errors.

- [x] A non-modal Settings window (tray → Settings…, ⌘, while the menu is open; `settings_window.rs`, objc2-app-kit) with a control for every setting except `excluded_devices`, which the tray's Microphones menu edits: General, Shortcut, Popup, Feedback, Automatic mute, and External control, which warns about `allow_url_unmute`. Each change is saved and applied like a tray toggle, and the window shows changes from the tray, About → Reset Settings, and the settings reload.
- [x] A shortcut recorder: Change… pauses the mute hotkey (`Shortcuts::suspend` / `resume`; a push-to-talk hold ends with a mute) and takes the window's keys through a local `NSEvent` monitor. It accepts what `parse_shortcut` accepts except ⇧ or ⌥ alone on a key that types or edits text, says inline why it refuses a key, and stops on Esc, Cancel, another control, focus loss, closing, or 30 s without a key press. A shortcut macOS can't register leaves the one in use, and the window says why.
- [x] Item 7's errors: while settings.json is invalid, the top of the window shows the error with its path, line, and column, and an Open settings.json button (About's Open Settings is renamed to match). Changes still apply but aren't saved, as with the tray.
- [ ] Verify with the installed app: each control saves and shows changes made elsewhere; recording, including a shortcut another app holds, Esc, and closing the window or switching apps mid-recording; the invalid-file notice; ⌘W, ⌘,, and the edit shortcuts. Not run yet.

### 13. Event-driven device updates

CoreAudio property listeners (`AudioObjectAddPropertyListenerBlock`) on the device list and each input device's mute and volume close the ≤ 200 ms gap and cut polling cost; keep a slower poll as a safety net.

- [x] Listeners on the system object's device list and on each input device's native mute, `VolumeScalar`, and `'vmvc'`, those it has: added on hot-plug, removed on unplug, and all removed before the exit restore (`MicController::start_listening`, `sync_listeners`, `stop_listening`). They use `AudioObjectAddPropertyListener` with a function pointer, not the block API, which would need `block2` and `dispatch2` as direct dependencies and the same block again for each removal; a leaked `DeviceChangeSignal` is the client data, so a callback that races its removal stays harmless.
- [x] The callback only sets an atomic and posts `Message::DevicesChanged`, once per burst (`device_changes.rs`). Control changes count only while mute is enforced, so automatic gain control in a call doesn't wake the loop while the user talks. The loop polls at once, at most every 50 ms, so another app fighting the mute can't spin it.
- [x] The poll runs every 1 s while every listener is registered, and every 200 ms while one is missing. Every poll retries a missing listener; failures go through `FailureLog`, and the loop logs each interval change. The popup follows the cursor's monitor every 200 ms only while it may be on screen. FakeBackend records registrations and reports changes as CoreAudio does.
- [ ] Verify with the installed app: an external unmute is undone at once, hot-plugging, the idle cost, and a quit that leaves no listener behind. Not run yet.

### 14. Tray improvements

- [x] A template (monochrome) tray icon that adapts to light and dark menu bars, red only when muted; the white non-template icon was invisible on a light menu bar. The unmuted icon is now a black template image, set together with its flag (`set_icon_with_as_template`).
- [x] A device submenu listing each input device and its state (muted, volume fallback, skipped, failed), plus "Open Sound Settings". The tray's Microphones submenu has a check item per connected input device with its state (`MicController::device_states`), the excluded devices that aren't connected, and Open Sound Settings… (Sound → Input). It is combined with #19: a row's check mark is its exclusion.
- [x] A camera-in-use indicator in the tray (it showed only in the popup): a dot in the icon's top-right corner, in the icon's color, and a "Camera in use" tooltip line.
- [x] An optional right-click toggle, `right_click_toggles_mute` (default `false`: both clicks open the menu, as before). With it on, a left-click still opens the menu, and a right-click mutes or unmutes when the button is released, once per click, through `request_mic` with `MicCommand::Toggle`, as the Mute item does (`tray::command_for_click`). tray-icon's `menu_on_right_click` follows the setting from `UI::new`, `apply_settings`, and `UI::apply_window_change`; the Settings window has a checkbox under General, and About → Reset Settings turns it off. tray-icon 0.26 highlights the icon on a right press that opens no menu and never clears that highlight, so the toggle clears it. The loop now drains `TrayIconEvent::receiver()` on every pass, which also stops that unbounded queue's growth: tray-icon queued every click and every move of the pointer over the icon, and nothing read them.
- [ ] Verify the right-click toggle with the installed app: with the setting off, both clicks open the menu; with it on, a left-click opens the menu and a right-click toggles once, with the Mute item's sound, notification, and "Mic on" flash, and leaves the icon unhighlighted; turning the setting on and off in the Settings window, in `settings.json`, and with About → Reset Settings applies from the next click; in push-to-talk mode, a right-click's unmute lasts until the shortcut's next release. Not run yet.

### 15. Launch at login via `SMAppService`

Replace the hand-written LaunchAgent with `SMAppService.mainApp` (macOS 13+): it appears under System Settings → Login Items, survives moving the app, and has no stale path.

- [x] On macOS 13+ (runtime version check), register `SMAppService.mainApp`; keep the LaunchAgent on macOS 11–12, and on 13+ as the fallback when registering fails. Startup mirrors the login item's status into `launch_at_login` and replaces an earlier version's LaunchAgent with the login item (the plist stays if registering fails). While macOS waits for approval, the setting stays on and the tray item says where to approve; a dev run changes nothing.
- [x] `install.sh` and `uninstall.sh`: keep repointing an existing LaunchAgent; on 13+, say how the login item is managed and removed.
- [ ] Verify on macOS 13+ with the installed, ad-hoc signed app: toggling on and off (System Settings → General → Login Items), migration from an existing plist, a logout and login, and whether a login item still opens the app after `task install` re-signs it (reports for ad-hoc builds disagree). Not run yet.
- [x] `apply_settings` pushes `launch_at_login` to macOS after every settings save (`ui.rs::set_launch_at_login`), but only startup mirrors macOS into the setting. A login item the user removed in System Settings is registered again by the next unrelated save (e.g. toggling Show Popup). Mirror the OS state before pushing, or push only when the setting itself changed. `UI::set_launch_at_login` now pushes only a value that differs from the one macOS last took or that startup mirrored from it (`launch_at_login::should_push`), so a settings reload after a hand edit or About → Reset Settings leaves the login item alone. The tray toggle still registers and unregisters; a failed push is retried by the next settings change. Limit: while the app runs, a removal in System Settings isn't reflected until the next launch, so the tray item stays checked until then.

### 16. Automatic mute triggers

Optional settings: mute on launch or restore the last state; mute on screen lock, sleep, or lid close (`NSWorkspace` and distributed notifications). Never unmute automatically.

- [x] `launch_state`: `"keep"` (default, today's behavior: adopt the hardware state), `"muted"`, or `"last"`. "Last" can only mute: it mutes when the user last chose mute, else keeps. The choice is recorded after every request in `mute-state.json` next to `settings.json` (`mute_triggers::LastMuteState`, through `store.rs`), never in `settings.json`.
- [x] `mute_on_lock` (screen lock or a switch to another user), `mute_on_sleep`, `mute_on_display_sleep`, all `false` by default, through #11's `session_events.rs` observer. Nothing unmutes on unlock or wake.
- [x] Lid close has no public notification. Without an external display it sleeps the Mac and its display, which `mute_on_sleep` and `mute_on_display_sleep` cover; in clamshell mode nothing reports it, and nothing mutes. Documented in `docs/settings.md`.

### 17. External control through the URL scheme

`com.brettinternet.mic-mute://toggle|mute|unmute|status`. The scheme is already registered (`Cargo.toml:66`) but unhandled; handling it enables Shortcuts.app, Raycast, Stream Deck, and scripts.

- [x] Handle tao's `Event::Opened`. A launch by URL can deliver the URL before `StartCause::Init`, so actions wait in a bounded queue (`url_scheme::UrlActions`) until the UI exists.
- [x] Any app or web page can open the URL, so only `mute` and `status` run by default; `unmute` and `toggle` need `allow_url_unmute` (default `false`) and are otherwise refused at `warn!`. Unknown actions and malformed URLs are logged and ignored. Accepted actions log at `info!`; no log line shows a query or fragment.
- [x] `status` can't return a value, so it shows the popup with the current state for 2 s, also while `show_popup` is off.
- [x] README: Shortcuts.app, Raycast, and `open` in scripts.

### 18. Feedback options

An optional sound and/or macOS notification on toggle; popup position, size, and "Mic on" flash duration (it was fixed at 1 s).

- [x] `play_sound` (default `false`): the system sound Bottle on mute, Tink on unmute (`NSSound soundNamed:`), on the output device without blocking the loop. Only a request that changes the state plays one (`feedback::change_to_report`); the poll's re-mutes, a mute while muted, and a failed change play nothing.
- [x] `show_notification` (default `false`): "Mic off" / "Mic on" through `UNUserNotificationCenter` (objc2-user-notifications; `NSUserNotification` is deprecated), for the same changes, each replacing the last. Outside an app bundle the framework aborts the process, so only the installed app posts; a dev run logs a warning. Ad-hoc signing should suffice: macOS keys the permission on the bundle id, which the installed app's signature identifier matches (see Verify). Focus and screen sharing can hide them, and no banner shows while Mic Mute is the active app.
- [x] `popup_position` (default `"bottom_center"`, where the popup has always been; it was never at the top): six positions on the cursor's monitor, 40 pt from its edges and below its menu bar.
- [x] `popup_size` (default `"medium"`, 250 × 40 pt): `small` is 0.75×, `large` 1.5×; text, icons, and spacing scale with it, and the popup's icons now render at 2 px per SVG unit, sharp at every size.
- [x] `popup_mic_on_ms` (default `1000`): `0` never shows "Mic on"; values above `10000` are capped.
- [x] An unknown or out-of-range value of the three popup settings falls back with a `warn!` instead of invalidating the file (`settings::Lenient`) and is saved back unchanged. About → Reset Settings restores all five settings.
- [ ] Verify with the installed app: each sound from the shortcut and the tray, none from the poll's re-mutes; the notification prompt and a notification per change; each position and size on one and two monitors, a notched menu bar included; flashes of `0` and `3000` ms. Not run yet.

### 22. Transient mute popup

With `show_popup` on, the popup stayed on screen for as long as the microphones were muted. An option shows it only briefly after each mute, so no popup stays on screen.

- [x] `popup_mic_off_ms` (default `null`: the popup stays up while muted, as before; a number hides it that many ms after each mute, capped at `10000` like `popup_mic_on_ms`), read through `settings::Lenient` (`MicOffFlash`): a bad value falls back to `null` with a warning and is saved back unchanged. Every mute request shows "Mic off" and starts the time over, automatic mutes included (launch, `mute_on_*`, a push-to-talk release); the poll's re-mutes and camera changes only redraw, without raising the popup; an unmute shows "Mic on" for `popup_mic_on_ms`, also after the popup hid; a `status` URL still shows it for 2 s. The popup keeps its flash deadlines (`popup::FlashDeadlines`, pure and tested with synthetic instants), which the loop folds into `WaitUntil` through `UI::popup_hide_at`, so no request path can leave it on screen; the cursor is tracked only while the popup may be on screen. Settings window: Hide “Mic off” after, with a millisecond field and stepper enabled only while it's checked; the note rows are now found by their views (`row_index`). About → Reset Settings restores `null`.
- [ ] Verify with the installed app, with `popup_mic_off_ms` at `1500` and `show_popup` on: a mute from the shortcut, the tray, and a `mute` URL each shows "Mic off" for 1.5 s, and a second mute during it restarts the time; a push-to-talk hold shows "Mic on", its release "Mic off"; with the popup hidden, another app's unmute (re-muted by the poll) and a camera turning on show nothing; an unmute after "Mic off" hid shows "Mic on" for `popup_mic_on_ms`; a `status` URL shows the popup for 2 s while muted, then hides it; the popup follows the cursor to another monitor during a flash; `0` shows nothing on a mute; the Settings window's checkbox, field, and stepper; turning the setting on while muted hides the popup, and turning it off shows it again. Not run yet.

### 19. Per-device exclusions

- [x] Upstream PR #8 (`excluded_devices`, an "Excluded Mics" submenu) handles virtual devices that accept mute but stay live, such as Teams Audio. Merge it when upstream lands it, or port it. Ported in the fork's own design: `excluded_devices` entries are keyed by UID with a display name (upstream's bare UID strings still load), excluding a device undoes what the app changed on it by the journal, and the Microphones submenu from #14 edits the list without PR #8's confirmation dialog. About → Reset Settings empties it. Merging PR #8 itself would now conflict.

### 20. Platform modernization

- [x] Migrate `cocoa` / `objc` / `block` to `objc2` + `objc2-app-kit`; typed `msg_send!` removes item 10's ABI-width bug class and the `block` future-incompatibility warning.
  - Migrated: the popup, its content, About (its button target is a `define_class!` class), and the mute sounds (objc2-app-kit and objc2-foundation: typed methods and constants, `Retained`, `MainThreadMarker`); the session event observer (`define_class!`, typed NSWorkspace and distributed center calls), camera detection (objc2-av-foundation, one device array per check instead of two), Open Sound Settings (`NSWorkspace::openURL`), the Dock toggle (`NSApplication::setActivationPolicy` behind a `MainThreadMarker`), and the camera thread's autorelease pool. `cocoa`, `objc`, and `#[macro_use] extern crate objc` are gone, and with them `block` 0.1's future-incompatibility warning.
- [x] Upgrade tao, resvg, and tray-icon + muda together (tray-icon 0.26 needs muda 0.21 and Rust 1.90). Now tao 0.37, tray-icon 0.26, muda 0.21, resvg 0.48; global-hotkey 0.8 is current. One version each of objc2 and the objc2-* crates; keyboard-types is in the tree twice (0.7 for global-hotkey, 0.8 for muda), so the tray accelerator converts the shortcut by key name. tray-icon 0.26 draws an icon at its pixel size up to 22 pt instead of 18 pt, so the tray icon renders at 36 px and the tray sets its image back to 18 pt, sharp on a 2x display; muda 0.21 rejects the empty key that cleared the Mute item's shortcut label, so the item is re-added instead.
- [ ] Verify the upgrade with the installed app: the tray icon's size and tint on light and dark menu bars, its menu on left and right click, the Mute item's shortcut label, the popup on each monitor, Show in Dock on and off, and a launch by URL. Not run yet.
- [x] Universal binary (arm64 + x86_64) for Intel Macs. Releases (`release.yaml`, `task release`) build both targets with `--locked`, combine them with `lipo`, check the slices with `lipo -archs`, and bundle the result (`target/universal/release/bundle/osx`); `task install -- --universal` does the same locally, while a plain `task install` stays native. CI also builds `x86_64-apple-darwin`, and `deny.toml` checks both targets.
- [x] Declare `rust-version` in `Cargo.toml`: 1.90, which tray-icon 0.26 and muda 0.21 need (the crate itself needs 1.88).
- [x] Add a `rust-toolchain.toml`: 1.96.0 with rustfmt, clippy, and both macOS targets. The workflows install it with `rustup toolchain install` instead of a toolchain action, and `mise.toml` reads it instead of installing `latest`.
- [x] Unified logging (OSLog), per the TODO at `main.rs:15`. Every line that passes `RUST_LOG` goes to stderr and to the unified log under the subsystem `com.brettinternet.mic-mute`, with one category per log target, so Console and `log stream` show the installed app's lines; a panic's message reaches it too. It's a minimal FFI to `_os_log_impl`, checked against clang's `os_log` buffer on both architectures. The `oslog` crate hasn't been released since 2022, needs a C build script (`cc`), and maps levels differently.
- [ ] Verify with the installed app: its lines in Console and `log stream` (`--level debug`), debug lines after `open --env RUST_LOG=debug -a "Mic Mute"`, and a launch without them. Not run yet.
