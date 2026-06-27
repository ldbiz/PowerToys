# Important Power Display files

## Module entry points

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplayModuleInterface/dllmain.cpp` | Native PowerToys module implementation. | Shows enable/disable, launch actions, hotkey metadata, GPO checks, events, and how the WinUI process is controlled from the runner side. |
| `src/modules/powerdisplay/PowerDisplayModuleInterface/PowerDisplayProcessManager.cpp` | Starts/stops `PowerToys.PowerDisplay.exe` and opens the named pipe. | Explains process lifetime and the one-way runner-module-to-app message channel. |
| `src/modules/powerdisplay/PowerDisplay/Program.cs` | WinUI process entry point. | Shows GPO early exit, single-instance behavior, logger setup, runner PID/pipe-name parsing, and app startup. |
| `src/modules/powerdisplay/PowerDisplay/PowerDisplayXAML/App.xaml.cs` | WinUI app launch orchestration. | Registers named events, starts named-pipe processing, monitors the runner process, creates the main window, and initializes the tray icon. |

## UI/frontend

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplay/PowerDisplayXAML/MainWindow.xaml(.cs)` | Flyout window UI host. | Connects window behavior, hotkeys, monitor cards, and identification UI to the main view model. |
| `src/modules/powerdisplay/PowerDisplay/ViewModels/MainViewModel.cs` | Core UI state and construction. | Creates `MonitorManager`, loads settings/profiles, starts initial discovery, and wires display-change watching. |
| `src/modules/powerdisplay/PowerDisplay/ViewModels/MainViewModel.Monitors.cs` | Monitor discovery orchestration. | Calls `MonitorManager.DiscoverMonitorsAsync`, rebuilds monitor view models, filters hidden monitors, and handles refresh failure state. |
| `src/modules/powerdisplay/PowerDisplay/ViewModels/MainViewModel.LinkedBrightness.cs` | Linked brightness behavior. | Explains the “All Displays” brightness slider and how linked monitor writes are planned. |
| `src/modules/powerdisplay/PowerDisplay/ViewModels/MonitorViewModel.cs` | Per-monitor UI state and commands. | Slider changes become brightness/contrast/volume writes through `MonitorManager`; unsupported features are hidden or disabled here. |

## Backend/service logic

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplay/Helpers/MonitorManager.cs` | Central monitor manager. | Routes discovery to WMI and DDC/CI, stores current monitor lookup, selects the controller for writes, and normalizes operation errors. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Services/MonitorStateManager.cs` | Persisted monitor state. | Supports restore-on-startup and profile/state behavior that can trigger hardware writes after discovery. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Services/ProfileService.cs` | Profile persistence. | Applies saved monitor setting sets, including brightness, to currently discovered monitor IDs. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Services/LinkedBrightnessPlanner.cs` | Linked-brightness planning. | Determines which monitors receive linked brightness changes and what should be saved. |

## Models/contracts

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplay.Lib/Models/Monitor.cs` | Runtime monitor model. | Contains monitor ID, communication method, current brightness, VCP maxima, capability flags, display name, GDI device name, and WMI instance name. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Models/MonitorOperationResult.cs` | Operation success/failure result. | Carries write failures from controller code back to manager/view model logging. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Models/MonitorIdentity.cs` | Monitor ID normalization. | Maps QueryDisplayConfig device paths and WMI instance names to a shared ID; identity mismatches are central to VDI diagnosis. |
| `src/modules/powerdisplay/PowerDisplay.Models/ProfileMonitorSetting.cs` | Persisted per-monitor profile values. | Shows which hardware values profiles may apply after discovery. |

## Display/monitor integration

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DisplayConfigInventory.cs` | Active display inventory. | Uses DisplayConfig APIs to produce the base list of displays before WMI/DDC routing. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/WMI/WmiController.cs` | WMI brightness controller. | Uses WMI brightness classes to discover, read, and write brightness for displays exposed by WMI. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DDC/DdcCiController.cs` | DDC/CI controller. | Enumerates monitor handles, retrieves physical monitor handles/capabilities, reads VCP 0x10 brightness, and writes VCP values. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DDC/PhysicalMonitorHandleManager.cs` | Physical monitor handle cache. | Important for any path requiring physical monitor handles rather than just logical display presence. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Utils/MccsCapabilitiesParser.cs` | Capabilities parsing. | Determines which DDC/CI VCP features are considered supported. |

## Settings/configuration

| File / directory | Role | Why it matters |
|---|---|---|
| `src/settings-ui/Settings.UI.Library/PowerDisplaySettings.cs` | Main settings class. | Defines module name, settings version, hotkey accessor, and retention behavior. |
| `src/settings-ui/Settings.UI.Library/PowerDisplayProperties.cs` | User-configurable properties. | Holds monitor list, restore options, linked brightness, visibility toggles, max compatibility mode, and other settings consumed by the app. |
| `src/settings-ui/Settings.UI/ViewModels/PowerDisplayViewModel.cs` | Settings UI page model. | Persists settings, sends module events, reloads monitor metadata, handles enable warning/crash lock, and exposes user options. |
| `src/modules/powerdisplay/PowerDisplay/ViewModels/MainViewModel.Settings.cs` | Runtime application of settings. | Applies settings changes from Settings UI, persists monitor metadata, and applies profiles. |

## Logging/telemetry/error handling

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplay.Lib/Services/CrashDetectionScope.cs` | DDC discovery crash marker. | Creates discovery lock around DDC capability probing so later startup can detect likely crash-prone hardware. |
| `src/modules/powerdisplay/PowerDisplay.Lib/Services/CrashRecovery.cs` | Startup crash recovery. | Can auto-disable/flag Power Display after an orphaned discovery lock. |
| `src/modules/powerdisplay/PowerDisplay/Telemetry/Events/` | Telemetry events. | Records module start and settings telemetry, not detailed per-API diagnostics. |
| `src/modules/powerdisplay/PowerDisplayModuleInterface/Trace.cpp` | Native ETW trace events. | Captures enable and activation events from the module DLL. |

## Tests/mocks

| File / directory | Role | Why it matters |
|---|---|---|
| `src/modules/powerdisplay/PowerDisplay.Lib.UnitTests/` | Unit tests for library behavior. | Covers identity, linked brightness planning/settings, crash recovery/scope, blacklist, settings rebuild, and capability parsing. |
| `src/modules/powerdisplay/PowerDisplay.Lib.UnitTests/MonitorIdentityTests.cs` | Monitor ID tests. | Relevant to WMI/DisplayConfig matching and display identity mismatch questions. |
| `src/modules/powerdisplay/PowerDisplay.Lib.UnitTests/MccsCapabilitiesParserTests.cs` | DDC capabilities parser tests. | Relevant to unsupported-feature and capability-detection behavior. |
| `src/modules/powerdisplay/PowerDisplay.Lib.UnitTests/LinkedBrightnessPlannerTests.cs` | Linked brightness tests. | Verifies which monitors receive linked brightness changes. |
