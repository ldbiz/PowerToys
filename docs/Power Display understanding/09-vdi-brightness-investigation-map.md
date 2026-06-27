# VDI brightness investigation map

This map does not propose a fix. It identifies where to inspect and what evidence to collect when brightness control fails in VDI-like environments even though normal monitor settings are accessible.

## What the code appears to depend on for monitor brightness control

Power Display brightness control depends on a display passing these stages:

1. It appears in the active DisplayConfig inventory.
2. It is not filtered by the built-in blacklist.
3. It is claimed by WMI brightness discovery, or it remains for DDC/CI discovery.
4. It becomes a `Monitor` with a stable `Monitor.Id` and `CommunicationMethod`.
5. It reports/supports brightness:
   - WMI: via `WmiMonitorBrightness` / `WmiMonitorBrightnessMethods`.
   - DDC/CI: via MCCS capabilities or max-compat VCP probing including brightness VCP `0x10`.
6. A brightness write uses the same selected controller and monitor identity.

## Where physical monitor assumptions may enter

The DDC/CI path is the clearest place where physical-monitor assumptions enter the design:

- `DdcCiController` starts with logical monitor handles from `EnumDisplayMonitors`.
- It obtains physical monitor handles through DDC/CI helper/handle-manager code.
- It fetches MCCS capabilities and VCP feature values from the physical monitor handle.
- It writes brightness as VCP `0x10`.

The code does not say how VDI drivers behave here. It only shows that a Windows-visible display must provide this DDC/CI path to be controlled as a DDC/CI monitor.

## Where logical display/device identity is mapped to brightness-capable monitor objects

| Stage | Files | What to inspect |
|---|---|---|
| Active display inventory | `DisplayConfigInventory.cs` | Whether QueryDisplayConfig returns active paths, GDI device names, friendly names, and device paths. |
| Canonical monitor ID | `MonitorIdentity.cs` | Whether device paths and WMI instance names normalize to the same ID. |
| WMI pairing | `WmiController.cs` | Whether `WmiMonitorBrightness.InstanceName` entries match active DisplayConfig IDs. |
| DDC routing | `MonitorManager.cs` | Whether displays unclaimed by WMI are routed to DDC/CI. |
| DDC handle matching | `DdcCiController.cs`, `MonitorDiscoveryHelper.cs`, `PhysicalMonitorHandleManager.cs` | Whether logical monitor handles map to target GDI names and physical monitor handles. |
| Capability assignment | `DdcCiController.cs`, `MccsCapabilitiesParser.cs`, `Monitor.cs` | Whether brightness support is set for the monitor. |

## Where unsupported or failed brightness APIs are handled

- DisplayConfig failures return an empty inventory and log warnings/errors.
- `MonitorManager.SafeDiscoverAsync` logs controller discovery exceptions and returns an empty controller result.
- WMI read failures log warnings and return invalid brightness values; WMI write failures return `MonitorOperationResult.Failure` with classified text for some WMI HRESULTs.
- DDC capability failures can lead to missing monitors or missing feature flags; max compatibility mode may probe VCP features if enabled.
- `MonitorManager.ExecuteMonitorOperationAsync` returns failures for missing monitor, missing controller, controller failure, or exceptions.
- `MonitorViewModel.ApplyPropertyToHardwareAsync` logs failed brightness writes but does not surface a detailed UI diagnostic in the inspected path.

## Where errors may be swallowed, logged, surfaced, or ignored

| Area | Behaviour |
|---|---|
| DisplayConfig inventory | Non-zero API results can return empty data; detailed warnings are logged for source/target device info failures. |
| WMI discovery | Per-object issues are handled inside discovery; unmatched WMI brightness instances are logged and skipped. |
| Controller discovery | Top-level controller exceptions are logged as warnings and treated as empty results. |
| DDC capabilities | Empty/unparseable capability data is logged; monitor may not be created unless capabilities/probing produce supported VCP codes. |
| Brightness write | Failure result is logged by the view model. The slider UI does not obviously expose the low-level error text to the user. |
| Crash-prone DDC discovery | Crash lock/flag can auto-disable or lock the Settings UI page after an orphaned lock is detected. |

## Files likely needed for VDI-specific diagnosis

1. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DisplayConfigInventory.cs`
2. `src/modules/powerdisplay/PowerDisplay.Lib/Models/MonitorIdentity.cs`
3. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/WMI/WmiController.cs`
4. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DDC/DdcCiController.cs`
5. `src/modules/powerdisplay/PowerDisplay.Lib/Drivers/DDC/PhysicalMonitorHandleManager.cs`
6. `src/modules/powerdisplay/PowerDisplay/Helpers/MonitorManager.cs`
7. `src/modules/powerdisplay/PowerDisplay/ViewModels/MonitorViewModel.cs`
8. `src/settings-ui/Settings.UI.Library/PowerDisplayProperties.cs`
9. Power Display logs under the module’s log path, especially DisplayClassification, WMI, DDC, and MonitorManager messages.

## Test coverage map for VDI-relevant scenarios

| Scenario | Current visible coverage |
|---|---|
| Virtual displays | No direct visible coverage. |
| No brightness capability | Parser and capability/value logic has unit coverage; full UI/discovery behavior for no brightness capability is not clearly covered. |
| API failure | Crash recovery and parser edge cases are covered; live DisplayConfig/WMI/DDC API failures are not broadly covered. |
| Monitor enumeration returning partial data | Identity/settings pieces have tests; end-to-end partial DisplayConfig/WMI/DDC enumeration is not visible. |
| Display identity mismatch | `MonitorIdentityTests` cover normalization; live mismatch between WMI and DisplayConfig is not fully covered. |
| Remote/VDI sessions | No direct visible coverage. |
| Multi-monitor setups | Linked brightness and identity tests cover parts of multi-monitor logic; hardware/API multi-monitor integration is not visible. |
| Permissions | WMI access-denied classification exists in code; permission matrix tests are not visible. |

## Likely diagnostic questions

- Which Windows/display APIs are actually returning data on the affected VDI machine: DisplayConfig, WMI brightness classes, logical monitor enumeration, or DDC/CI physical monitor APIs?
- Does the affected display become a Power Display `Monitor` at all? If yes, what are its `Id`, `CommunicationMethod`, `GdiDeviceName`, and `SupportsBrightness` values?
- Does the module require physical monitor handles or DDC/CI-style capability for that display, or is it expected to be claimed by WMI?
- What happens when Windows exposes a display in Settings but not through the API path Power Display uses?
- Is the VDI case treated as unsupported capability, failed controller discovery, empty monitor enumeration, hidden monitor settings, or a write failure?
- Are DisplayClassification, WMI, DDC, and MonitorManager logs clear enough to diagnose the affected user machine?
- Does the UI distinguish “no displays found” from “displays found but brightness unsupported” and “brightness write failed”?
- Do persisted monitor IDs from a prior physical session mismatch IDs from the VDI session?
- Is `MaxCompatibilityMode` relevant in the affected case, or does discovery fail before DDC capability parsing/probing?
- Does restore-on-startup or profile application attempt brightness writes that fail before the user manually moves a slider?

## Maintainer questions before attempting a fix

- What display inventory and controller route should be considered supported for VDI or remote sessions?
- Should Power Display expose diagnostic state in the UI for monitors that are visible but not brightness-capable by this module’s API paths?
- Should logs capture more structured per-display API results for DisplayConfig, WMI, and DDC/CI without leaking sensitive identifiers?
- Should a virtual/remote display be explicitly recognized as unsupported, or should another brightness API path be considered?
- What manual reproduction environment should be used to validate any future change: RDP, enterprise VDI, GPU-backed virtual display, or another setup?
