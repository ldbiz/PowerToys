# Configuration and environment

## Settings files/classes/models

| Area | Files/classes | Runtime consequence |
|---|---|---|
| Main module settings | `PowerDisplaySettings`, `PowerDisplayProperties` | Defines module name, hotkey, monitor list, feature visibility, restore settings, max compatibility mode, linked brightness, tray/options state. |
| Runtime settings application | `MainViewModel.Settings.cs` | Applies hidden monitor filtering, feature visibility, max compatibility mode, profiles, and telemetry. |
| Settings UI page | `PowerDisplayViewModel.cs`, `PowerDisplayPage.xaml(.cs)` | User changes settings, enable state, monitor options, profiles, and sends events to Power Display. |
| Monitor metadata | `MonitorInfo` in Settings UI library and `MonitorSettingsRebuilder` in PowerDisplay.Lib | Saves discovered monitor information so Settings UI can configure per-monitor options even after discovery. |
| Profiles | `PowerDisplayProfile`, `PowerDisplayProfiles`, `ProfileMonitorSetting`, `ProfileService` | Can apply saved brightness/contrast/etc. values to matching monitor IDs. |
| Monitor state | `MonitorStateManager`, `MonitorStateFile`, `MonitorStateEntry` | Supports restore-on-startup and saved current values. |

## Defaults and user-configurable options

Visible code shows these relevant defaults or settings concepts:

- `PowerDisplaySettings` uses module name `PowerDisplay`, version `1`, and `PowerDisplayProperties` defaults.
- Per-monitor settings entries are retained for 30 days after last successful discovery unless hidden.
- New monitor cards default brightness/contrast/volume visibility from detected support, while input source, power state, and color temperature are disabled by default in the flyout until enabled through settings.
- `MonitorRefreshDelay` controls display-change debounce delay and is clamped to 1–30 seconds in `MainViewModel`.
- `MaxCompatibilityMode` is pushed to the DDC/CI controller before discovery and can allow direct VCP probing when capability strings are missing or unusable.
- `RestoreSettingsOnStartup` can trigger monitor setting writes after initial discovery.
- `LinkedLevelsActive` and `ExcludedFromSyncMonitorIds` control linked brightness behavior.
- Hidden monitors are skipped when building the visible view model collection.

## PowerToys-wide settings dependency

Power Display uses `SettingsUtils.Default` and the PowerToys Settings UI repository infrastructure. The WinUI Power Display process reads settings directly, while Settings UI writes settings and signals the app to reload or rescan.

The native module also participates in PowerToys-wide module enablement. Settings UI updates `GeneralSettingsConfig.Enabled.PowerDisplay` for enable/disable unless GPO controls the state.

## Environment and capability assumptions visible in code

The code visibly depends on:

- The Power Display GPO value not being disabled at startup.
- The runner process existing when launched by PowerToys; the app exits if runner monitoring reports runner exit.
- Active display paths being returned by DisplayConfig APIs.
- WMI brightness classes existing for WMI-controlled internal panels.
- DDC/CI discovery obtaining logical monitor handles, physical monitor handles, and readable capabilities or successful max-compat probes for DDC-controlled monitors.
- User/process permission sufficient for WMI brightness methods and display APIs; WMI access-denied is explicitly classified as a possible failure.
- A writable local PowerToys data area for logs, settings, crash flags, profiles, and state files. DDC crash scope logs and proceeds if the lock cannot be written.

The code does not define VDI, remote desktop, or virtual-display-specific policy. Those environments must be diagnosed by observing how the same API paths behave there.
