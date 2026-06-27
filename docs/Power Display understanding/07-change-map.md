# Change map for future investigation

## If I need to understand module startup

- **Likely files/directories**: `PowerDisplayModuleInterface/dllmain.cpp`, `PowerDisplayProcessManager.cpp`, `PowerDisplay/Program.cs`, `PowerDisplay/PowerDisplayXAML/App.xaml.cs`.
- **Why those files matter**: They show enable/disable, process launch, GPO checks, single-instance behavior, event registration, named-pipe setup, and crash recovery ordering.
- **Caveats**: Startup success is separate from monitor API success. Do not infer brightness support from the app process being alive.

## If I need to understand monitor discovery

- **Likely files/directories**: `MonitorManager.cs`, `DisplayConfigInventory.cs`, `WmiController.cs`, `DdcCiController.cs`, `MonitorDiscoveryHelper.cs`, `PhysicalMonitorHandleManager.cs`, `MonitorBlacklistService.cs`.
- **Why those files matter**: They define the active display inventory, WMI-first routing, DDC/CI probing, physical handle use, and pre-controller filtering.
- **Caveats**: Discovery failures can be converted into empty lists. Logs are essential for distinguishing DisplayConfig, WMI, DDC, blacklist, and capability failures.

## If I need to understand brightness read/write

- **Likely files/directories**: `MonitorViewModel.cs`, `MainViewModel.LinkedBrightness.cs`, `MonitorManager.cs`, `WmiController.cs`, `DdcCiController.cs`, `Monitor.cs`, `VcpFeatureValue.cs`.
- **Why those files matter**: They cover slider commits, linked broadcasts, controller selection, WMI `WmiSetBrightness`, DDC VCP `0x10`, and percent/raw conversion.
- **Caveats**: The UI may hide brightness before write code is ever reached if capability detection does not set `SupportsBrightness`.

## If I need to understand UI/backend communication

- **Likely files/directories**: `App.xaml.cs`, `NamedPipeProcessor.cs`, `PowerDisplayProcessManager.cpp`, `PowerDisplayViewModel.cs`, `MainViewModel.Settings.cs`, `Constants` in shared interop.
- **Why those files matter**: They explain runner-to-app messages, Settings UI events, settings reload, rescan, hotkey refresh, telemetry, and profile application.
- **Caveats**: Brightness slider writes are not an external IPC call to the runner; they occur inside the WinUI process through `MonitorManager`.

## If I need to understand settings/configuration

- **Likely files/directories**: `PowerDisplaySettings.cs`, `PowerDisplayProperties.cs`, `PowerDisplayViewModel.cs`, `MainViewModel.Settings.cs`, `MonitorSettingsRebuilder.cs`, `ProfileService.cs`, `MonitorStateManager.cs`.
- **Why those files matter**: They govern hidden monitors, feature visibility, restore-on-startup, profiles, linked brightness, monitor metadata retention, and max compatibility mode.
- **Caveats**: Settings can affect whether a monitor/control is visible without changing OS-level brightness support.

## If I need to understand failures in VDI/virtual display environments

- **Likely files/directories**: `DisplayConfigInventory.cs`, `MonitorIdentity.cs`, `WmiController.cs`, `DdcCiController.cs`, `PhysicalMonitorHandleManager.cs`, `MonitorManager.cs`, `MonitorViewModel.cs`.
- **Why those files matter**: They are the points where a Windows-visible display must become an active inventory entry, a WMI or DDC/CI monitor, a brightness-capable model, and finally a write target.
- **Caveats**: The repo does not define how VDI display drivers behave. Treat VDI as an environment to observe through logs and API results, not as a known code path.

## If I need to add or adjust tests

- **Likely files/directories**: `PowerDisplay.Lib.UnitTests/`, especially identity, parser, linked brightness, blacklist, crash recovery, and settings rebuild tests.
- **Why those files matter**: Most existing tests target pure logic and are easier to extend without hardware.
- **Caveats**: Tests involving actual DisplayConfig/WMI/DDC APIs likely need abstraction, fixtures, or environment-specific integration testing. Avoid changing tests until the investigation identifies a concrete behavior to verify.
