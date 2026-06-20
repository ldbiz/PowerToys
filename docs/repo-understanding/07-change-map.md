# Change Map: Where to Look for Common Tasks

This document helps you quickly find the right files when making changes to PowerToys.

## If I Need to Change Module Behavior

### Core Module Lifecycle or Interface

**Files to check first**:
- `src/modules/interface/powertoy_module_interface.h` – The contract all modules implement
- `src/runner/powertoy_module.cpp` – Module loading and method invocation
- `src/runner/main.cpp` – Module enumeration and startup orchestration

**Why**: Any change to the module interface requires updating all modules that implement it. Changes to module loading affect all modules.

**Caveats**:
- Interface changes are breaking; coordinate with module maintainers
- Module loading path is performance-sensitive (runs on every startup)

### Specific Module Logic (e.g., Color Picker)

**Files to check first**:
- `src/modules/<modulename>/` – Module source directory
- Module's `dllmain.cpp` or main module file – Entry point and interface implementation
- Module's `<ModuleName>Module.cpp` – Core logic

**Why**: Module-specific logic is isolated in the module's directory.

**Caveats**:
- Each module may have its own architecture (some use external apps, some are pure DLL)
- Some modules have corresponding WinUI/WPF UIs in parallel directories

### Module Hotkey Handling

**Files to check first**:
- Module's `get_hotkeys()` implementation – Declares which hotkeys the module uses
- Module's `on_hotkey()` implementation – Handles hotkey presses
- `src/runner/centralized_kb_hook.cpp` – Global hotkey registration (if modifying hook itself)

**Why**: Hotkey flow is critical for performance; changes here affect user experience.

**Caveats**:
- `on_hotkey()` runs in a low-level keyboard hook; must be fast (< 10 ms)
- Changes to hotkey registration should be tested on actual systems with external apps using conflicting hotkeys

## If I Need to Change the Settings UI or User Experience

### Main Settings UI

**Files to check first**:
- `src/settings-ui/Settings.UI/MainWindow.xaml.cs` – Settings UI entry point
- `src/settings-ui/Settings.UI/SettingsXAML/` – XAML pages for each module's settings
- `src/settings-ui/Settings.UI.Library/` – Shared UI controls and view models

**Why**: Settings UI is the primary user-facing component; all user-facing changes flow through it.

**Caveats**:
- Settings UI is a separate WinUI 3 process; changes here don't affect Runner stability
- XAML binding changes must align with ViewModel implementation

### Quick Access Flyout

**Files to check first**:
- `src/settings-ui/QuickAccess.UI/` – Quick access UI project
- IPC communication in `src/common/interop/two_way_pipe_message_ipc.h`

**Why**: Quick access (left-click tray icon) shows a lightweight UI; changes here affect tray interaction.

**Caveats**:
- Quick access is separate from main Settings UI; maintain consistency

### Module Settings Pages

**Files to check first**:
- `src/settings-ui/Settings.UI/SettingsXAML/Panels/<Module>Panel.xaml` – Module's settings UI
- Corresponding `<Module>Panel.xaml.cs` code-behind
- `src/settings-ui/Settings.UI.Library/ViewModels/<Module>ViewModel.cs` – State management

**Why**: Each module's settings have dedicated UI; changes are isolated.

**Caveats**:
- ViewModels must communicate with Runner via IPC for persistence
- Settings changes trigger `set_config()` on the module; ensure module handles new/changed properties gracefully

## If I Need to Change Configuration Handling

### Settings File Format or Persistence

**Files to check first**:
- `src/common/SettingsAPI/settings_helpers.h/.cpp` – Settings file I/O
- `%LocalAppData%\Microsoft\PowerToys\settings.json` – The actual file (check format)
- Module's `get_config()` and `set_config()` implementations

**Why**: Settings are the persistence layer; changes here affect all modules.

**Caveats**:
- Settings file is versioned; breaking changes require migration logic
- Settings are JSON; ensure new properties have sane defaults

### IPC Message Format

**Files to check first**:
- `src/common/interop/two_way_pipe_message_ipc.h` – IPC transport
- `src/runner/main.cpp` – IPC message handler in Runner
- `src/settings-ui/Settings.UI/MainWindow.xaml.cs` – IPC client in Settings UI

**Why**: IPC is the communication channel between Runner and Settings UI; format changes are breaking.

**Caveats**:
- IPC message format is JSON; document the schema
- Ensure backward compatibility or add version handshake

### Enterprise Policy (GPO) Integration

**Files to check first**:
- `src/common/utils/gpo.h` – GPO query utilities
- `src/common/GPOWrapper/` – GPO wrapper for managed code
- Module's use of GPO in `enable()` or `set_config()`

**Why**: GPO controls are enterprise-critical; changes can lock out IT admins.

**Caveats**:
- GPO keys are read from registry; ensure key paths follow Microsoft conventions
- Restricted settings should prevent user modification (read-only UI)

## If I Need to Change the Tray Icon or System Integration

### System Tray Icon

**Files to check first**:
- `src/runner/tray_icon.cpp/.h` – Tray icon implementation
- `src/runner/main.cpp` – Tray icon initialization
- `src/runner/tray_icon_window_proc()` – Message handling

**Why**: Tray icon is the primary user interface to PowerToys.

**Caveats**:
- Tray icon runs in the main Runner thread; changes affect responsiveness
- Message handling must be fast; avoid blocking operations

### Runner Auto-Start and Installation

**Files to check first**:
- `installer/` – WiX installer projects
- `src/runner/auto_start_helper.cpp` – Auto-start registry setup
- `src/runner/general_settings.cpp` – Startup behavior

**Why**: Installation and auto-start are separate from runtime behavior.

**Caveats**:
- Installer changes affect all deployments; test thoroughly
- Auto-start logic interacts with Windows Startup folder and registry

## If I Need to Change Hotkey Registration or Detection

### Hotkey Conflict Detection

**Files to check first**:
- `src/runner/hotkey_conflict_detector.cpp` – Conflict detection logic
- `src/runner/centralized_hotkeys.cpp` – Central hotkey registry
- Settings UI's hotkey validation (probably in `src/settings-ui/Settings.UI.Library/`)

**Why**: Hotkey conflicts cause user confusion; detection prevents conflicts when possible.

**Caveats**:
- Conflict detection can't prevent all conflicts (external apps may use same hotkey)
- Detecting conflicts with arbitrary apps is unreliable; focus on known Microsoft hotkeys

### Global Keyboard Hook

**Files to check first**:
- `src/runner/centralized_kb_hook.cpp` – Hook installation and callback
- `src/runner/centralized_hotkeys.cpp` – Hotkey matching
- `src/runner/main.cpp` – Hook cleanup on shutdown

**Why**: The keyboard hook is the core mechanism for capturing global hotkeys.

**Caveats**:
- Low-level hook is performance-sensitive; callback must be fast
- Hook is per-process; affects all modules
- Some antivirus software may interfere with hook installation

## If I Need to Change Logging or Telemetry

### Logging Configuration and Output

**Files to check first**:
- `src/common/logger/` – Logger implementation
- `src/runner/main.cpp` – Logger initialization
- Modules' `#include <common/logger/logger.h>` usage

**Why**: Logging is essential for debugging; consistent configuration ensures usable logs.

**Caveats**:
- Log file location is per-user AppData
- Logging level affects performance; keep hot paths silent (INFO or ERROR only)

### Telemetry Events

**Files to check first**:
- `src/common/Telemetry/` – Telemetry infrastructure
- `src/common/Telemetry/TraceBase.h` – ETW event definition
- Modules' telemetry calls (usually in module's `enable()`, `on_hotkey()`, etc.)

**Why**: Telemetry tracks feature adoption and crashes.

**Caveats**:
- Telemetry is opt-in; don't assume all users enable it
- No PII should ever be logged; all events are anonymized

## If I Need to Change Build Configuration or Dependencies

### Project Structure and Build

**Files to check first**:
- `PowerToys.slnx` – Solution file grouping all projects
- `Directory.Build.props` – Global MSBuild properties
- `tools/build/build.ps1` – Build orchestration script
- Individual `.csproj` and `.vcxproj` files

**Why**: Build configuration affects all modules.

**Caveats**:
- Solution structure is complex (30+ modules); changes may have wide impact
- Build scripts use PowerShell; careful with special characters and paths

### Dependencies and NuGet Packages

**Files to check first**:
- `nuget.config` – NuGet feed configuration
- Individual `.csproj` – Package references
- `vcpkg.json` – C++ dependencies

**Why**: Dependencies are managed per-project; adding new deps affects licensing and security.

**Caveats**:
- All new dependencies must be audited for security and license compliance
- Updating dependencies can introduce breaking changes; test thoroughly

### Code Style and Formatting

**Files to check first**:
- `src/.editorconfig` – Formatting rules (C++, C#, XAML)
- `CppRuleSet.ruleset` – C++ static analysis rules
- `doc/devdocs/development/style.md` – Coding conventions

**Why**: Style consistency makes code reviews easier.

**Caveats**:
- EditorConfig rules are enforced by CI; violations prevent merge
- Different languages (C++, C#) have different style guides

## If I Need to Change Module Loading or the Plugin System

### Module Discovery and Loading

**Files to check first**:
- `src/runner/main.cpp` – Module list initialization
- `src/runner/powertoy_module.cpp` – Module loading and wrapper
- `src/modules/interface/powertoy_module_interface.h` – Interface contract

**Why**: Module loading is fundamental to the entire system.

**Caveats**:
- Changing module loading affects all modules
- DLL loading is order-sensitive; some modules depend on others being loaded first

### Module Enable/Disable Lifecycle

**Files to check first**:
- Individual module's `enable()` and `disable()` implementations
- `src/runner/main.cpp` – Lifecycle orchestration
- `src/common/SettingsAPI/settings_helpers.h` – Enabled state persistence

**Why**: Enable/disable is the primary way users control modules.

**Caveats**:
- `disable()` must free all resources (hooks, threads, timers)
- `enable()` must be idempotent (calling twice should be safe)

## Summary: Quick Lookup

| Change Type | Primary Directory | Key File(s) |
|-------------|-------------------|-------------|
| Module behavior | `src/modules/<name>/` | `dllmain.cpp` |
| Settings UI | `src/settings-ui/` | `MainWindow.xaml.cs`, Panels |
| Tray icon | `src/runner/` | `tray_icon.cpp` |
| Hotkey handling | `src/runner/` | `centralized_kb_hook.cpp`, `centralized_hotkeys.cpp` |
| Settings persistence | `src/common/` | `SettingsAPI/settings_helpers.h` |
| IPC communication | `src/common/interop/` | `two_way_pipe_message_ipc.h` |
| Build system | Root, `tools/build/` | `PowerToys.slnx`, `build.ps1` |
| Logging | `src/common/logger/` | `logger.h` |
| Telemetry | `src/common/Telemetry/` | `TraceBase.h` |
| Enterprise policy | `src/common/` | `utils/gpo.h` |

Start with the "Primary Directory" and key files; they will guide you to related code.
