# Power Display understanding: summary

## What Power Display does

Power Display is a PowerToys module for viewing and controlling display-related monitor settings from a small WinUI flyout/tray application. In the inspected code, its main visible controls are monitor brightness, contrast, volume, color preset, input source, power state, rotation, profiles, monitor identification, and linked brightness across multiple brightness-capable displays.

For the VDI investigation, the most relevant behaviour is that brightness is not controlled through a single generic “Windows Settings” path. Power Display builds its own monitor inventory, routes displays to either WMI or DDC/CI controllers, and then issues brightness read/write calls through the controller selected during discovery.

## How it fits into PowerToys

Power Display has two parts:

- `PowerDisplayModuleInterface`, a native module DLL loaded by the PowerToys runner.
- `PowerToys.PowerDisplay.exe`, a WinUI 3 process launched by the module DLL and kept running while the module is enabled.

The native module handles PowerToys runner integration, enable/disable, hotkey metadata, launch actions, and named-pipe/event signaling. The WinUI process owns the actual display discovery and monitor control UI.

## Runtime model relevant to this module

Power Display is a long-running side process rather than only an in-runner module. The runner module starts the WinUI process with the runner PID and a generated named-pipe name. The app also registers named Windows events for toggling, settings refresh, hotkey refresh, rescans, telemetry, Light Switch profile application, and termination.

Monitor discovery happens in the WinUI process, primarily through `MainViewModel` and `MonitorManager`. The manager builds a DisplayConfig inventory, asks WMI first which displays expose WMI brightness, and sends the remaining active displays to DDC/CI discovery.

## Main entry points

| Entry point | Role |
|---|---|
| `src/modules/powerdisplay/PowerDisplayModuleInterface/dllmain.cpp` | PowerToys module implementation loaded by the runner. |
| `src/modules/powerdisplay/PowerDisplayModuleInterface/PowerDisplayProcessManager.cpp` | Starts/stops `PowerToys.PowerDisplay.exe` and writes messages to its named pipe. |
| `src/modules/powerdisplay/PowerDisplay/Program.cs` | WinUI process entry point, single-instance setup, GPO check, logger initialization. |
| `src/modules/powerdisplay/PowerDisplay/PowerDisplayXAML/App.xaml.cs` | App launch, crash recovery gate, event registration, named-pipe client, tray icon, main window creation. |
| `src/modules/powerdisplay/PowerDisplay/ViewModels/MainViewModel*.cs` | UI state, monitor discovery orchestration, settings/profile application, linked brightness. |
| `src/modules/powerdisplay/PowerDisplay/Helpers/MonitorManager.cs` | Unified monitor inventory and control operations. |

## Main internal components

- **Module process management**: runner-loaded DLL starts and stops the WinUI process.
- **UI/view models**: `MainWindow`, `MainViewModel`, and `MonitorViewModel` expose controls and bind slider actions to monitor operations.
- **Monitor discovery**: `DisplayConfigInventory`, `WmiController`, and `DdcCiController` produce `Monitor` objects.
- **Monitor identity**: `MonitorIdentity` normalizes Windows display paths and WMI instance names into the module’s monitor IDs.
- **Settings/profiles/state**: `PowerDisplaySettings`, profiles, and monitor state files persist module choices and monitor snapshots.
- **Failure/crash handling**: logging, operation result objects, DDC crash detection/recovery, and Settings UI crash lock state.

## External OS/display APIs and abstractions visible in code

Power Display uses these visible API paths or abstractions:

- DisplayConfig APIs through `DisplayConfigInventory` (`GetDisplayConfigBufferSizes`, `QueryDisplayConfig`, `DisplayConfigGetDeviceInfo`) to enumerate active display paths.
- WMI classes `root\WMI:WmiMonitorBrightness` and `WmiMonitorBrightnessMethods` for internal-panel brightness discovery/read/write.
- Win32 monitor APIs such as `EnumDisplayMonitors`, `GetMonitorInfo`, and physical monitor/DDC functions wrapped by `DdcCiNative` and `PhysicalMonitorHandleManager` for DDC/CI-managed monitors.
- `ChangeDisplaySettingsEx` through `DisplayRotationService` for rotation, separate from brightness.
- WinRT `DisplayMonitor` device watcher plus power setting notifications for display-change rescans.

## Read these first

1. `src/modules/powerdisplay/PowerDisplay/Helpers/MonitorManager.cs`
2. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DisplayConfigInventory.cs`
3. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/WMI/WmiController.cs`
4. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DDC/DdcCiController.cs`
5. `src/modules/powerdisplay/PowerDisplay.Lib/Models/Monitor.cs`
6. `src/modules/powerdisplay/PowerDisplay.Lib/Models/MonitorIdentity.cs`
7. `src/modules/powerdisplay/PowerDisplay/ViewModels/MainViewModel.Monitors.cs`
8. `src/modules/powerdisplay/PowerDisplay/ViewModels/MonitorViewModel.cs`
9. `src/modules/powerdisplay/PowerDisplayModuleInterface/dllmain.cpp`
10. `src/settings-ui/Settings.UI.Library/PowerDisplaySettings.cs`

## Why this pack exists

This pack is scoped to investigating why Power Display brightness control may fail in VDI, virtual display, remote desktop, or non-standard display-driver environments even when ordinary Windows monitor settings appear accessible. It documents what the repository actually shows: the discovery, identity mapping, controller selection, brightness read/write paths, and places where failures are logged, surfaced, or reduced to empty/unsupported monitor state.
