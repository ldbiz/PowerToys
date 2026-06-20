# Core Concepts

## Modules

**Meaning**: Self-contained utility component loaded and managed by the PowerToys Runner. Each module is a DLL implementing the PowertoyModuleIface interface.

**Where it appears**: 
- Loaded from `src/modules/*/` directories
- DLL files loaded at runtime by `src/runner/powertoy_module.cpp`
- Enumerated in module list maintained by Runner

**Related files**: 
- `src/modules/interface/powertoy_module_interface.h` (contract)
- `src/runner/powertoy_module.cpp` (loader)
- `src/runner/main.cpp` (module list initialization)

**Lifecycle**: Created on Runner startup, enabled/disabled via Settings UI, destroyed on Runner shutdown.

---

## Hotkey

**Meaning**: A keyboard shortcut (key + modifiers like Ctrl, Shift, Alt, Win) that a module registers to receive events from the Runner when pressed globally.

**Structure**: Defined in `PowertoyModuleIface::Hotkey` with fields for modifiers (win, ctrl, shift, alt), the key code, and a unique ID within the module.

**Where it appears**:
- Declared by modules in `get_hotkeys()` method
- Registered with Runner in `src/runner/centralized_kb_hook.cpp`
- Triggered via low-level keyboard hook (`SetWindowsHookEx`)
- Routed to module's `on_hotkey()` method

**Related files**:
- `src/modules/interface/powertoy_module_interface.h`
- `src/runner/centralized_kb_hook.cpp` (hook installation)
- `src/runner/centralized_hotkeys.cpp` (hotkey conflict detection)

---

## Module Interface

**Meaning**: The C++ abstract class contract that every PowerToy module must implement to be compatible with the Runner.

**Key methods**:
- `get_key()` – Return module's unique identifier
- `enable()`/`disable()` – Start/stop the module
- `get_config()` – Return JSON describing available settings
- `set_config()` – Apply user configuration changes
- `on_hotkey()` – Handle registered hotkey press
- `destroy()` – Clean up resources

**Where it appears**:
- Defined in `src/modules/interface/powertoy_module_interface.h`
- Implemented by all modules (e.g., `src/modules/colorPicker/ColorPickerModule/dllmain.cpp`)
- Called by Runner in `src/runner/powertoy_module.cpp`

**Related files**:
- `src/modules/interface/powertoy_module_interface.h`
- `tools/project_template/ModuleTemplate` (template implementation)

---

## Named Pipes (IPC)

**Meaning**: The Windows inter-process communication mechanism used for bidirectional JSON message exchange between Runner and Settings UI, and between modules and the Settings UI.

**Transport**: `TwoWayPipeMessageIPC` class handles connection setup, message framing, and async reading/writing.

**Message format**: JSON strings sent over named pipes, e.g., `{"ShowYourself":"Dashboard"}`.

**Where it appears**:
- Pipe server (Runner-side) in `src/runner/main.cpp` listening for Settings UI connection
- Pipe client (Settings UI-side) in `src/settings-ui/Settings.UI/MainWindow.xaml.cs`
- Bidirectional communication for module settings queries and updates

**Related files**:
- `src/common/interop/two_way_pipe_message_ipc.h/.cpp`
- `doc/devdocs/core/runner.md` (IPC communication section)

---

## Settings

**Meaning**: User-configurable preferences for the overall PowerToys system and individual modules, stored as JSON files in the user's AppData folder.

**Structure**: 
- Global settings (e.g., enabled modules list)
- Per-module settings (e.g., Color Picker's auto-copy behavior)
- Stored in `%LocalAppData%/Microsoft/PowerToys/settings.json`

**Access pattern**: 
- Runner and modules read settings via `SettingsAPI` helpers
- Settings UI writes settings through IPC, which triggers module `set_config()` calls
- Settings are versioned to support migration between versions

**Where it appears**:
- Stored in `src/common/SettingsAPI/settings_helpers.h`
- Accessed by modules in their `get_config()` and `set_config()` implementations
- Displayed and edited in `src/settings-ui/`

**Related files**:
- `src/common/SettingsAPI/settings_helpers.h`
- `doc/devdocs/core/settings/project-overview.md`

---

## Group Policy (GPO)

**Meaning**: Windows enterprise policy mechanism allowing IT administrators to control PowerToys behavior across networked machines without per-user intervention.

**Use cases**: 
- Enable/disable specific modules
- Restrict certain settings (read-only for users)
- Enforce default configurations

**Where it appears**:
- Registry keys read by modules during startup
- GPO wrapper in `src/common/GPOWrapper/`
- DSC (Desired State Configuration) support in `src/dsc/`

**Related files**:
- `src/common/utils/gpo.h`
- `src/dsc/` (Desired State Configuration manifests)
- `doc/devdocs/core/settings/gpo-integration.md`

---

## Telemetry

**Meaning**: Event Tracing for Windows (ETW) mechanism for collecting anonymized usage data about module activation, crashes, and feature adoption.

**Privacy model**: User opt-in; data does not include personal files or content; all events are logged to local ETW trace (can be inspected locally before upload).

**Where it appears**:
- Traced events in module initialization, hotkey triggers, errors
- ETW provider configured in `src/common/Telemetry/`
- Telemetry headers in module implementations

**Related files**:
- `src/common/Telemetry/TraceBase.h`
- `src/common/Telemetry/ProjectTelemetry.h`
- Telemetry calls embedded in module code

---

## DPI Awareness

**Meaning**: The system's ability to scale UI and calculations correctly on high-DPI displays (e.g., 150%, 200% scaling).

**Implementation**:
- Monitor DPI values fetched via `GetDpiForMonitor()`
- UI elements scaled proportionally
- Calculations adjust for DPI to prevent distortion

**Where it appears**:
- Multi-monitor utilities (FancyZones, MouseUtils) heavily use DPI logic
- `src/common/Display/dpi_aware.h` provides utilities
- `src/common/Display/DisplayUtils.h` enumerates monitors with DPI

**Related files**:
- `src/common/Display/dpi_aware.h`
- `src/common/Display/DisplayUtils.h`

---

## Shell Extensions

**Meaning**: Modules that integrate with Windows File Explorer by adding context menu items (right-click menu) or preview handlers.

**Examples**: 
- PowerRename (context menu for batch rename)
- Preview Pane handlers (custom file preview)

**Registration**: Done via Windows Registry; modules declare the handler CLSID and file types they support.

**Where it appears**:
- Registry registration in installer (`src/modules/*/installer/`)
- COM interface implementation in module DLL
- Context menu strings localized in resource files

**Related files**:
- `src/modules/powerrename/` (shell extension example)
- `src/modules/previewpane/` (preview handler example)

---

## Module State and Enabled/Disabled

**Meaning**: Modules have runtime state (enabled or disabled) that is persisted in settings and controls whether the module's functionality is active.

**Lifecycle**:
- Module is instantiated (constructor)
- If settings indicate enabled, `enable()` is called (hooks registered, listeners set up)
- User changes setting → `set_config()` called, which may trigger `disable()` then `enable()` if key config changes
- On app exit, `disable()` then `destroy()` called

**Where it appears**:
- Settings UI toggles enable/disable for each module
- Runner calls `enable()`/`disable()` methods
- `is_enabled()` query used to determine if module should process hotkeys

**Related files**:
- `src/runner/main.cpp` (lifecycle orchestration)
- Module implementations (e.g., `src/modules/colorPicker/ColorPickerModule/dllmain.cpp`)

---

## External Application Launcher

**Meaning**: A module that does not implement all functionality directly, but instead launches a separate executable (often a C# WinUI or WPF application) to handle the actual user interaction.

**Pattern**:
- Module DLL registers hotkey
- When hotkey pressed, module launches external app (e.g., Color Picker UI)
- External app and module may communicate via IPC or named pipes

**Examples**: 
- Color Picker (launches UI for color selection)
- Advanced Paste (launches UI for paste options)

**Where it appears**:
- Module DLL in `src/modules/<name>/`
- External UI project alongside (e.g., `src/modules/colorPicker/ColorPickerUI/`)

**Related files**:
- `src/modules/colorPicker/ColorPickerModule/` and `ColorPickerUI/`

---

## Summary

These concepts form the vocabulary of the PowerToys architecture:
- **Modules** are the building blocks (plugins)
- **Hotkeys** are how users trigger them
- **Settings** store configuration, persisted to disk
- **IPC (Named Pipes)** connects processes for control and configuration
- **GPO** enables enterprise policy control
- **Telemetry** tracks usage (opt-in, privacy-respecting)
- **Shell Extensions** integrate with Windows File Explorer
- **DPI Awareness** ensures correct scaling on high-DPI displays

Understanding these concepts makes it clear how PowerToys code is organized and how changes propagate through the system.
