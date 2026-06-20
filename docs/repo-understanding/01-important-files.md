# Important Files and Directories

## Core Runtime

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/runner/main.cpp` | PowerToys.exe entry point | Controls module lifecycle, tray icon, hotkey dispatch—the heart of the system |
| `src/runner/tray_icon.cpp/.h` | Tray icon implementation | Handles user interaction (clicks, menu), triggers Settings UI display |
| `src/runner/powertoy_module.cpp/.h` | Module loader and wrapper | Loads DLLs, caches module metadata, calls module methods |

## Module Interface and Contracts

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/modules/interface/powertoy_module_interface.h` | Module contract definition | Every module must implement this; defines get_key(), enable(), disable(), set_config(), on_hotkey() |
| `src/modules/interface/` | Module interface implementations | Declarations for hotkey structures, lifecycle states |

## Settings UI

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/settings-ui/Settings.UI/MainWindow.xaml.cs` | Settings UI entry point | WinUI 3 host; initiates IPC connection to Runner |
| `src/settings-ui/Settings.UI.Library/` | Shared UI logic and view models | Reusable controls, module settings pages, and data binding |
| `src/settings-ui/QuickAccess.UI/` | Quick access flyout | Lightweight UI shown on left-click of tray icon |

## IPC and Communication

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/common/interop/two_way_pipe_message_ipc.h/.cpp` | Named pipe IPC transport | Bidirectional JSON message channel between Runner and Settings UI |
| `src/common/SettingsAPI/settings_helpers.h` | Settings persistence | Reads/writes module settings from/to JSON files in user AppData |

## Common Libraries

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/common/logger/` | Logging infrastructure | spdlog wrapper for C++; centralized log files and debug output |
| `src/common/Telemetry/` | ETW-based telemetry | Event tracing for usage analytics (opt-in, privacy-respecting) |
| `src/common/Display/` | Monitor and DPI utilities | DisplayUtils, monitors.h; needed by multi-monitor modules |
| `src/common/utils/` | General utilities | JSON parsing, string handling, registry access, path manipulation |
| `src/common/notifications/` | Windows notification helpers | Toast notifications for user alerts |

## Example Modules

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/modules/colorPicker/` | Color picker utility | Example of external launcher module (C#-based UI) |
| `src/modules/powerrename/` | Batch file rename utility | Example of shell extension module (context menu integration) |
| `src/modules/fancyzones/` | Window layout manager | Example of complex module with real-time hooks and editor UI |
| `src/modules/cmdpal/` | Command palette | Extensible module with extension SDK; shows advanced IPC |

## Build Configuration

| File / Directory | Role | Why It Matters |
|---|---|---|
| `PowerToys.slnx` | Solution file | Top-level Visual Studio project grouping all modules and components |
| `tools/build/build.ps1` | Build orchestrator | Main build script; handles platform/config selection and project discovery |
| `tools/build/build-common.ps1` | Build helpers | Shared build logic for different project types |
| `Directory.Build.props` | Global MSBuild properties | Sets version, common flags, output paths for all projects |
| `src/.editorconfig` | Code style enforcement | StyleCop rules for C#, formatting rules for C++ |

## Installation and Packaging

| File / Directory | Role | Why It Matters |
|---|---|---|
| `installer/` | WiX installer projects | MSI/MSIX generation; defines registry keys, shortcuts, file layout |
| `installer/PowerToysSetup/` | Main installer | Coordinates module installation, desktop shortcut creation |

## Documentation and Development Guidance

| File / Directory | Role | Why It Matters |
|---|---|---|
| `doc/devdocs/core/architecture.md` | Architecture overview | Module types, Runner design, IPC patterns |
| `doc/devdocs/core/runner.md` | Runner internals | Detailed lifecycle, tray icon message handling, hotkey dispatch |
| `doc/devdocs/core/settings/readme.md` | Settings system documentation | Settings architecture, view models, data flow |
| `doc/devdocs/development/logging.md` | Logging conventions | How to use spdlog and Logger in modules |
| `doc/devdocs/development/style.md` | Coding style guide | C++, C#, XAML formatting and naming conventions |

## Tests

| File / Directory | Role | Why It Matters |
|---|---|---|
| `src/common/UnitTests-CommonLib/` | Common library tests | Tests for shared utilities, JSON parsing, IPC |
| `src/modules/*/unittests/` or `*/tests/` | Module-specific tests | Each major module has unit tests (GoogleTest for C++, XUnit for C#) |
| `src/settings-ui/Settings.UI.UnitTests/` | Settings UI tests | ViewModel and control tests |

## Key Configuration Files

| File / Directory | Role | Why It Matters |
|---|---|---|
| `nuget.config` | NuGet package source configuration | Controls NuGet feed locations and authentication |
| `vcpkg.json` | C++ dependency manifest | vcpkg dependencies (boost, win32metadata, etc.) |
| `.github/` | GitHub-specific configuration | CI/CD workflows, issue templates, branch protection |

---

## Summary

The most critical paths to understand are:
1. **Runner startup** – `src/runner/main.cpp` and module loading
2. **Module interface** – `src/modules/interface/powertoy_module_interface.h`
3. **Settings communication** – IPC in `src/common/interop/` and Settings UI entry
4. **Build system** – `tools/build/build.ps1` and MSBuild structure

Everything else flows from understanding how these core pieces interact.
