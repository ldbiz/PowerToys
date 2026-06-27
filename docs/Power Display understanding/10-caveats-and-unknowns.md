# Caveats and unknowns

## Missing context

- This review used repository code and existing repo-understanding docs, not a live Windows machine with monitors attached.
- The actual generated settings JSON on a user machine was not inspected.
- Power Display logs from the affected VDI environment were not available.
- The precise VDI product, display driver model, session type, and Windows build are unknown.

## Existing repo-understanding docs that appeared stale or too general

The existing `docs/repo-understanding/` pack is useful as a broad PowerToys overview, but it is too general for Power Display diagnosis. Some examples are not Power Display-specific and may not apply directly:

- The general pack emphasizes runner hotkey dispatch, while Power Display comments indicate its hotkey handling is in-process in `PowerToys.PowerDisplay.exe` rather than via runner hotkey dispatch.
- The general pack describes broad environment variables and registry keys that were not confirmed in the Power Display code inspected for this task.
- The general pack does not describe Power Display’s DisplayConfig → WMI/DDC routing, physical monitor handles, VCP capabilities, crash recovery, or linked brightness behavior.

No direct contradiction was found in the sense of a Power Display-specific statement being false; the issue is that the general pack is not granular enough for this module.

## External OS/display API behaviour not confirmable from repo

The repository shows which APIs Power Display calls. It does not confirm how these APIs behave in:

- VDI environments.
- Remote desktop sessions.
- Virtual display adapters.
- GPU remoting products.
- Non-standard display drivers.
- Systems where Windows Settings can show brightness but WMI/DDC/CI paths differ.

The code documents dependencies on DisplayConfig, WMI brightness classes, and DDC/CI monitor handles/capabilities. It does not prove which dependency fails in a given VDI setup.

## Behaviour visible in code but not covered by visible tests

- End-to-end DisplayConfig inventory enumeration.
- WMI brightness discovery/read/write against real WMI providers.
- DDC/CI physical monitor handle acquisition and VCP read/write against real monitors.
- UI slider debounce interacting with real hardware writes.
- Error presentation in the flyout for brightness write failure.
- Display-change watcher behaviour in virtual or remote sessions.
- Full app startup from runner module DLL through named pipe and events.

## Behaviour inferred from tests but not clearly visible as integration

- Identity parsing is well represented by unit tests, but live pairing between WMI and DisplayConfig depends on actual OS strings.
- Capability parsing is tested, but complete DDC discovery depends on handle retrieval and native API responses.
- Linked brightness planning is tested, but successful hardware writes for all linked targets depend on per-monitor controller behaviour.

## Questions for a human maintainer

- Is Power Display intended to support VDI/remote/virtual display brightness at all, or should those environments be explicitly unsupported?
- Are there existing user logs from VDI failures showing DisplayClassification, WMI, or DDC messages?
- Should the UI expose diagnostic differences between no monitor, no brightness capability, and brightness write failure?
- Are there privacy constraints around logging full monitor device paths or should logs use redacted IDs/EDID segments only?
- Is `MaxCompatibilityMode` intended only for physical DDC/CI monitors, or should it be presented in diagnosis for virtual/remote displays?
- Is there a preferred OS API for brightness in the specific VDI environment that differs from both WMI brightness and DDC/CI?

## VDI-specific assumptions that cannot be confirmed from code

- Whether the affected VDI exposes active DisplayConfig target device paths.
- Whether it exposes `WmiMonitorBrightness` and `WmiMonitorBrightnessMethods` for the virtual display.
- Whether `EnumDisplayMonitors` returns handles that can be mapped to DisplayConfig GDI names in the session.
- Whether physical monitor handles exist or DDC/CI VCP operations are meaningful in the session.
- Whether Windows Settings uses the same API path as Power Display for brightness in that environment.
- Whether permissions/elevation differ between local and VDI sessions for WMI brightness methods.
