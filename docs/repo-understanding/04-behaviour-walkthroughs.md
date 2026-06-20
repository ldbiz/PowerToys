# Behavior Walkthroughs

## Behavior 1: User Enables a Module via Settings UI

**Trigger**: User opens PowerToys Settings UI → clicks toggle to enable a previously disabled module (e.g., Color Picker).

### Files Involved

- `src/settings-ui/Settings.UI/MainWindow.xaml.cs` – Settings UI entry, IPC connection
- `src/common/interop/two_way_pipe_message_ipc.h/.cpp` – Named pipe communication
- `src/runner/main.cpp` – Receives IPC message, calls module methods
- Module's `dllmain.cpp` or equivalent – `enable()` and `set_config()` implementations

### Flow

1. **Settings UI captures toggle change**: User toggles enable/disable in the UI for a module.
2. **Settings UI sends IPC message**: Constructs JSON containing the module key and new enabled state, sends over the named pipe to the Runner.
3. **Runner receives message**: Settings UI's named pipe listener callback in Runner triggers.
4. **Runner updates settings file**: The enable/disable state is written to `%LocalAppData%\Microsoft\PowerToys\settings.json`.
5. **Runner calls module.enable()**: If transitioning to enabled state, the Runner calls the module's `enable()` method.
6. **Module registers hotkeys**: The module's `enable()` method sets up its hotkeys by returning them from `get_hotkeys()`.
7. **Runner registers hotkeys**: The Runner adds the returned hotkeys to its centralized keyboard hook.
8. **Module is now active**: The module's `on_hotkey()` method will now be called when the user presses its hotkey.

### Inputs & Outputs

- **Input**: User toggle in Settings UI
- **Output**: Module is now listening for hotkeys; module state persisted to disk

### External Calls / Side Effects

- File write to settings.json
- Keyboard hook update (adds new hotkey)
- Module may allocate resources (threads, timers) in `enable()`

### Tests Covering This

- Settings UI unit tests (likely in `src/settings-ui/Settings.UI.UnitTests/`)
- Module enable/disable unit tests (module-specific test projects)
- Integration tests for IPC communication (if present)

---

## Behavior 2: User Presses a Hotkey and Module Launches External UI

**Trigger**: User presses a registered hotkey (e.g., Win+Shift+C for Color Picker).

### Files Involved

- `src/runner/centralized_kb_hook.cpp` – Low-level keyboard hook installation and callback
- `src/runner/hotkey_conflict_detector.cpp` – Checks for conflicting hotkeys
- Module's `dllmain.cpp` – `on_hotkey()` implementation
- External application (e.g., `src/modules/colorPicker/ColorPickerUI/`) – Launched by module

### Flow

1. **User presses hotkey**: Keyboard input captured by Windows OS.
2. **Low-level hook called**: Windows calls the callback registered in `SetWindowsHookEx()` in `centralized_kb_hook.cpp`.
3. **Hook translates keystroke**: Converts the raw key event into a hotkey structure (win/ctrl/shift/alt + key code).
4. **Hotkey matched**: Compares against the list of registered hotkeys from all enabled modules.
5. **Module identified**: Determines which module registered this hotkey and which hotkey ID within that module.
6. **Module.on_hotkey() called**: Runner calls the module's `on_hotkey()` method with the hotkey ID.
7. **Module launches UI process**: Module's `on_hotkey()` typically creates a new process (e.g., ColorPickerUI.exe).
8. **External UI runs**: The external application performs the user-facing task (e.g., color selection).
9. **UI communicates result**: External app may communicate back to the module via IPC or file (e.g., copied color to clipboard).
10. **UI closes**: User finishes task; external app exits.
11. **Runner continues**: Returns to message loop, ready for next hotkey press.

### Inputs & Outputs

- **Input**: User keystroke
- **Output**: External UI displayed; action completed (e.g., color copied to clipboard)

### External Calls / Side Effects

- System keyboard hook callback
- Process creation (external UI executable)
- May modify clipboard, registry, or file system
- Telemetry event logged

### Tests Covering This

- Keyboard hook unit tests
- Module's `on_hotkey()` tests (usually mock the process launch)
- Integration tests (if any) that simulate hotkey press and verify expected action

---

## Behavior 3: Settings JSON File Is Updated and Module Config Changes

**Trigger**: User modifies a module's setting in the Settings UI (e.g., Color Picker's "auto-copy" toggle).

### Files Involved

- `src/settings-ui/Settings.UI/` – Module settings page (XAML + ViewModel)
- `src/common/interop/two_way_pipe_message_ipc.h` – IPC transport
- `src/runner/main.cpp` – IPC message handler
- `src/common/SettingsAPI/settings_helpers.h` – Settings persistence
- Module's `dllmain.cpp` – `set_config()` implementation

### Flow

1. **User changes setting**: Modifies a toggle, slider, or text field in the module's settings panel.
2. **Settings UI updates ViewModel**: The XAML binding updates the backing data model.
3. **Settings UI sends IPC message**: Constructs JSON with the module key and updated config object, sends over the named pipe.
4. **Runner receives message**: The message pump in Runner processes the IPC callback.
5. **Runner parses JSON**: Extracts module ID and new configuration.
6. **Runner updates settings file**: Calls `SettingsAPI` to write the new config to `settings.json`.
7. **Runner calls module.set_config()**: Passes the JSON configuration string to the module.
8. **Module updates internal state**: Parses the JSON and applies settings (e.g., disables auto-copy behavior).
9. **Module may trigger enable/disable**: If the change requires rebuilding hotkeys or state, the module may internally call `disable()` then `enable()`.
10. **Result persisted**: New setting is now stored on disk; will be reloaded on PowerToys restart.

### Inputs & Outputs

- **Input**: User's setting change in UI
- **Output**: Module behavior changed; new configuration persisted to disk

### External Calls / Side Effects

- File I/O to settings.json
- Named pipe IPC
- Module internal state change (may affect subsequent hotkey presses)

### Tests Covering This

- Settings file I/O tests
- Module `set_config()` unit tests
- IPC communication tests
- Settings UI viewmodel tests

---

## Behavior 4: PowerToys Starts Up and Modules Load from Disk

**Trigger**: User launches PowerToys.exe or OS auto-starts it.

### Files Involved

- `src/runner/main.cpp` – Startup orchestration
- `src/runner/powertoy_module.cpp` – Module DLL loading
- `src/common/SettingsAPI/settings_helpers.h` – Reads settings.json
- `src/common/logger/` – Logging subsystem
- Module DLLs (e.g., `src/modules/colorPicker/ColorPickerModule.dll`)

### Flow

1. **PowerToys.exe launched**: User double-clicks shortcut or OS auto-starts it.
2. **Logger initialized**: Logging system set up; logs written to file on disk.
3. **Single-instance mutex checked**: Ensures only one PowerToys process runs; if another exists, this process exits.
4. **Common utilities initialized**: String tables, COM, WinRT initialization.
5. **Settings.json read**: Runner loads global settings (list of enabled modules, etc.).
6. **Tray icon created**: Window class registered with OS; tray icon appears in system tray.
7. **Keyboard hook installed**: Low-level keyboard hook via `SetWindowsHookEx()`.
8. **Module enumeration**: Runner iterates over known module DLL paths.
9. **Module DLLs loaded**: For each module, `LoadLibrary()` called; module is now in memory.
10. **powertoy_create() called**: Runner calls the exported function to instantiate the module.
11. **Module object stored**: Module added to Runner's module list.
12. **For each enabled module**:
    - `get_config()` called to read persisted settings
    - `enable()` called to start the module
    - `get_hotkeys()` called to retrieve hotkeys
    - Hotkeys registered with the keyboard hook
13. **Message loop entered**: Runner now waits for keyboard events, tray events, or IPC messages.
14. **Modules ready**: All enabled modules are active and will respond to their hotkeys.

### Inputs & Outputs

- **Input**: PowerToys.exe executable, settings.json file from disk
- **Output**: Running PowerToys process with all enabled modules loaded and hotkeys registered

### External Calls / Side Effects

- File I/O (read settings.json, open log file)
- DLL loading (module DLLs)
- Windows API calls (keyboard hook, tray icon registration)
- Telemetry may be logged

### Tests Covering This

- Module loading unit tests
- Integration tests that verify module enumeration
- Settings loading tests
- Startup sequence tests (if present)

---

## Behavior 5: Hotkey Conflict Detected and User Notified

**Trigger**: User tries to assign a hotkey that conflicts with another application's hotkey (e.g., Win+A already used by Windows Search).

### Files Involved

- `src/runner/hotkey_conflict_detector.cpp` – Conflict detection logic
- `src/runner/centralized_hotkeys.cpp` – Central hotkey registry
- Settings UI validation logic
- Windows Registry or external API queries

### Flow

1. **User attempts to assign hotkey**: In Settings UI, selects a hotkey combination.
2. **Settings UI queries Runner**: Sends IPC message asking if hotkey is available or used.
3. **Runner checks global hotkey registry**: Queries `centralized_hotkeys.cpp` to see if any PowerToys module already uses it.
4. **Conflict detection runs**: Calls `hotkey_conflict_detector.cpp` to check if Windows or other apps claim the hotkey.
5. **Conflict status returned**: IPC response indicates conflict or no conflict.
6. **Settings UI displays warning**: If conflict detected, shows toast or dialog notifying user of the conflict.
7. **User is given choice**: Can accept the conflict (may not work as intended) or select a different hotkey.
8. **If user proceeds**: New hotkey assigned; if conflict, the actual hotkey press may be intercepted by the other app.

### Inputs & Outputs

- **Input**: User attempts to set a hotkey
- **Output**: Conflict warning displayed; hotkey assigned (may not function if truly conflicted)

### External Calls / Side Effects

- Registry queries (to check Windows reserved hotkeys)
- Toast notification to user
- Settings file updated with new (possibly conflicted) hotkey

### Tests Covering This

- Conflict detection unit tests (mocking registry queries)
- Integration tests simulating known conflicts
- Settings UI validation tests

---

## Summary

These five behaviors cover the key user interactions with PowerToys:
1. **Module enable/disable** – Lifecycle management through Settings UI
2. **Hotkey press** – Core user action triggering module functionality
3. **Settings change** – Configuration persistence and module reconfiguration
4. **Startup sequence** – Module loading and initialization
5. **Conflict detection** – Safeguard for hotkey collisions

Each involves multiple components working together through IPC, file I/O, and the keyboard hook. Understanding these flows makes it clear how changes in one component might affect others.
