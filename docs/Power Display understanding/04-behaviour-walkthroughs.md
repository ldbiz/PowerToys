# Behaviour walkthroughs

## 1. Module startup and initialization

| Item | Details |
|---|---|
| Trigger | PowerToys starts or the user enables Power Display. |
| Files/modules involved | `PowerDisplayModuleInterface/dllmain.cpp`, `PowerDisplayProcessManager.cpp`, `Program.cs`, `App.xaml.cs`, `MainViewModel.cs`. |
| Bird’s-eye flow | The native module starts the WinUI process, the app initializes logging and events, then constructs the main window/view model. The view model immediately starts monitor discovery and keeps UI interaction disabled while scanning. |
| Inputs | Runner enable state, GPO state, runner PID, generated pipe name, settings. |
| Outputs | Running `PowerToys.PowerDisplay.exe`, tray icon/window state, event listeners, initial monitor list. |
| External calls or side effects | Process launch via ShellExecute, named pipe creation/connection, Windows event listeners, telemetry start event, log files. |
| Tests that appear to cover it | Crash detection/recovery and settings-related logic have unit tests; full runner-to-WinUI startup is not covered by visible Power Display unit tests. |
| VDI notes | Startup success does not prove brightness API availability. Discovery can still produce empty or unsupported monitor state after the UI process starts. |

## 2. Discovering/listing displays and monitors

| Item | Details |
|---|---|
| Trigger | Initial view model construction, display-change watcher event, Settings UI rescan event, or explicit refresh. |
| Files/modules involved | `MainViewModel.Monitors.cs`, `MonitorManager.cs`, `DisplayConfigInventory.cs`, `WmiController.cs`, `DdcCiController.cs`, `MonitorIdentity.cs`. |
| Bird’s-eye flow | The manager reads active displays from DisplayConfig, filters built-in blacklisted EDID IDs, asks WMI which active displays expose brightness, then probes DDC/CI for remaining active displays. The view model filters hidden monitors and creates monitor cards. |
| Inputs | Active DisplayConfig paths, WMI brightness instances, logical monitor handles, DDC physical handles/capabilities, blacklist/settings. |
| Outputs | Sorted `Monitor` list and visible `MonitorViewModel` collection. |
| External calls or side effects | QueryDisplayConfig, DisplayConfigGetDeviceInfo, WMI queries, EnumDisplayMonitors, physical monitor and capabilities calls, logs, crash detection lock around DDC discovery. |
| Tests that appear to cover it | Identity mapping, blacklist behavior, capability parser, and monitor settings rebuilding are tested. End-to-end OS discovery is not visible in tests. |
| VDI notes | This is the key path to inspect when Windows Settings can see a display but Power Display shows none or shows it without brightness. |

## 3. Reading current brightness

| Item | Details |
|---|---|
| Trigger | Monitor discovery initializes monitor state; controllers also expose `GetBrightnessAsync`. |
| Files/modules involved | `WmiController.cs`, `DdcCiController.cs`, `Monitor.cs`, `MonitorViewModel.cs`. |
| Bird’s-eye flow | WMI reads `CurrentBrightness` for a matched WMI instance. DDC/CI initializes brightness from VCP `0x10` only when capabilities say brightness is supported, converting raw current/max to percent. The view model copies `Monitor.CurrentBrightness` into slider state. |
| Inputs | WMI instance name, DDC physical monitor handle, VCP capabilities, VCP current/max values. |
| Outputs | `Monitor.CurrentBrightness`, `MonitorViewModel.Brightness`, brightness slider availability. |
| External calls or side effects | WMI query, DDC VCP feature read, warning logs on invalid ranges or failures. |
| Tests that appear to cover it | `VcpFeatureValueTests` and parser tests cover value conversion/capability parsing. Direct WMI/DDC reads are not covered by visible tests. |
| VDI notes | A display may be enumerated but not expose WMI brightness or DDC VCP brightness, causing no brightness capability in the UI. |

## 4. Changing monitor brightness

| Item | Details |
|---|---|
| Trigger | Per-monitor brightness slider commit, profile application, restore-on-startup, or linked brightness broadcast. |
| Files/modules involved | `MonitorViewModel.cs`, `MainViewModel.LinkedBrightness.cs`, `MonitorManager.cs`, `WmiController.cs`, `DdcCiController.cs`, `MonitorStateManager.cs`, `ProfileService.cs`. |
| Bird’s-eye flow | UI changes debounce and call `SetBrightnessAsync`. The manager looks up the monitor by canonical ID, chooses WMI or DDC/CI based on `CommunicationMethod`, calls the controller write, updates in-memory state on success, and persists state through the view model. |
| Inputs | Monitor ID, requested brightness 0–100, selected controller, WMI instance name or DDC physical handle/max. |
| Outputs | Hardware/API write result, updated current brightness on success, saved monitor state/settings. |
| External calls or side effects | WMI `WmiSetBrightness` method or DDC VCP `0x10` write, logs, settings/state file writes. |
| Tests that appear to cover it | Linked brightness planning is tested; direct controller writes and UI slider-to-hardware integration are not visibly covered. |
| VDI notes | The most important log details are monitor ID, communication method, controller availability, and operation result/error message. |

## 5. Handling unsupported monitors or API failures

| Item | Details |
|---|---|
| Trigger | DisplayConfig failures, WMI query/method failures, DDC handle/capability failures, unsupported VCP codes, missing monitor lookup, or controller exceptions. |
| Files/modules involved | `DisplayConfigInventory.cs`, `MonitorManager.cs`, `WmiController.cs`, `DdcCiController.cs`, `MonitorViewModel.cs`, `CrashDetectionScope.cs`, `CrashRecovery.cs`. |
| Bird’s-eye flow | Discovery failures generally log and continue with fewer/no monitors. Unsupported DDC features remain absent from capability flags, which hides/disables UI controls. Write failures produce `MonitorOperationResult.Failure` and warnings/errors in the view model/manager. Crash-prone DDC discovery can be quarantined on the next startup. |
| Inputs | Win32/WMI error codes, empty API results, unsupported capabilities, exceptions. |
| Outputs | Empty monitor list, missing sliders, logged warnings/errors, failed operation results, possible crash lock/auto-disable marker. |
| External calls or side effects | Log files, crash flag/lock files, Settings UI crash lock state, no hardware state change on failure. |
| Tests that appear to cover it | Crash recovery/scope, blacklist, identity, linked settings, and parser behavior have tests. VDI-specific failure modes are not visible in tests. |
| VDI notes | The UI may not clearly distinguish every case: no displays, displays filtered/hidden, no brightness capability, and write failure are different code states that can look similar to users. |
