# PowerToys Repository Overview

## Purpose

Microsoft PowerToys is a Windows productivity utility suite that provides 30+ specialized tools to help power users customize and streamline everyday tasks. Each utility is a modular component that can be independently enabled/disabled through a centralized Settings UI.

Examples include FancyZones (window management), PowerRename (batch file renaming), Advanced Paste (smart clipboard), Color Picker, Command Palette, and many others.

## Technology Stack

| Layer | Technology |
|-------|-----------|
| **Runtime Executable** | C++ (PowerToys.exe runner) |
| **Module Interface** | C++ with COM/WinRT interop |
| **Settings UI** | WinUI 3 (C# XAML) + WPF |
| **Modules** | Mix of C++ (native hooks, performance-critical), C# (modern utilities), and shell extensions |
| **IPC** | Windows Named Pipes (JSON messages) |
| **Logging** | spdlog (C++), Logger (C#) |
| **Telemetry** | Event Tracing for Windows (ETW) |
| **Build System** | MSBuild, PowerShell scripts |
| **Installer** | WiX (MSI) + MSIX |

## Main Runtime Model

PowerToys operates as a tray-resident Windows application with two main processes:

1. **Runner (PowerToys.exe)** – Loads modules, handles hotkeys, manages lifecycle
2. **Settings UI** – Separate WinUI 3 application; communicates with Runner via named pipes

When the user presses a hotkey, the Runner intercepts it, identifies which module should handle it, and delegates the action. Each module can also launch external applications (e.g., Color Picker UI). Settings changes flow from Settings UI → Runner → Modules via the IPC pipe.

## Main Entry Points

| Entry Point | Path | Role |
|-------------|------|------|
| PowerToys Runner Executable | `src/runner/main.cpp` | System tray, module loader, hotkey dispatcher |
| Settings UI Application | `src/settings-ui/Settings.UI/MainWindow.xaml.cs` | Configuration UI for all modules |
| Module Interface | `src/modules/interface/powertoy_module_interface.h` | Contract all modules must implement |

## Architecture Summary

PowerToys follows a **plugin/module architecture** where the Runner acts as a host that loads and manages independent modules.

Each module is a DLL that implements a standardized interface defining:
- Module name and unique key
- Hotkey registration and handling
- Enable/disable lifecycle
- Configuration (JSON-based)
- Telemetry metadata

The Runner maintains a list of module DLLs, loads them on startup, and ensures only enabled modules are active. Modules are loosely coupled—they communicate with the Runner and Settings UI through named pipes, not directly with each other.

The Settings UI is a separate process that runs on-demand; it communicates with the Runner to persist user preferences and trigger module reconfigurations. This separation keeps the critical hotkey-dispatch path lightweight and reduces attack surface.

## Main External Services and Dependencies

- **Windows APIs** – Low-level keyboard/mouse hooks, shell extensions (COM), registry access
- **ETW** – Event Tracing for Windows for telemetry (opt-in)
- **Package Management** – NuGet (C#), vcpkg (C++)
- **Installer Platforms** – Microsoft Store (MSIX), WinGet, direct MSI downloads
- **DSC** – Desired State Configuration support for enterprise policy-driven installation

## Read These First

To understand the codebase efficiently, start with these files:

| File | Purpose |
|------|---------|
| `src/runner/main.cpp` | Entry point for PowerToys.exe; shows module loading and tray icon initialization |
| `src/modules/interface/powertoy_module_interface.h` | The contract every module must implement |
| `src/settings-ui/Settings.UI/MainWindow.xaml.cs` | Settings UI entry; shows IPC communication pattern |
| `doc/devdocs/core/architecture.md` | High-level architectural explanation |
| `doc/devdocs/core/runner.md` | Detailed Runner design and lifecycle |
| `tools/build/build.ps1` | Build entry point and configuration |
| `src/common/interop/two_way_pipe_message_ipc.h` | Named pipe IPC mechanism used throughout |
| `src/common/SettingsAPI/settings_helpers.h` | Settings storage and retrieval abstraction |

These files provide the foundation for understanding how modules load, how settings flow, how modules are enabled/disabled, and how the system handles hotkeys.
