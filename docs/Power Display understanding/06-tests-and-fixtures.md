# Tests and fixtures

## Test framework and structure

Power Display has a C# unit-test project under `src/modules/powerdisplay/PowerDisplay.Lib.UnitTests/`. The project name and test file style indicate library-level unit tests for Power Display’s pure logic and services. The top-level repo guidance says PowerToys tests are normally built first and run through Visual Studio Test Explorer or `vstest.console.exe`, not `dotnet test`.

## Important test files

| Test file | Area covered | Relevance |
|---|---|---|
| `MonitorIdentityTests.cs` | Canonical monitor ID parsing/matching. | Important for DisplayConfig/WMI identity mismatch diagnosis. |
| `MccsCapabilitiesParserTests.cs` | DDC/MCCS capability parsing. | Important for deciding whether brightness VCP `0x10` is considered supported. |
| `VcpFeatureValueTests.cs` | VCP value conversion/validity. | Relevant to DDC brightness current/max conversion. |
| `LinkedBrightnessPlannerTests.cs` | Linked brightness target planning. | Relevant when multiple monitors are brightness-capable. |
| `LinkedBrightnessSettingsTests.cs` | Persistence/settings for linked brightness. | Covers settings-driven linked behavior. |
| `MonitorSettingsRebuilderTests.cs` | Monitor settings metadata retention/rebuild. | Relevant to Settings UI monitor list and hidden/retained monitors. |
| `MonitorBlacklistServiceTests.cs` and `BuiltInMonitorBlacklistTests.cs` | Monitor blacklist behavior. | Relevant because blocked EDID IDs are filtered before controllers run. |
| `CrashDetectionScopeTests.cs` and `CrashRecoveryTests.cs` | DDC discovery crash lock/auto-disable behavior. | Relevant to crash-prone DDC capability probing. |
| `MonitorIdMigratorTests.cs`, `MonitorIdComparerTests.cs` | Legacy/current monitor ID behavior. | Relevant to persisted settings/profiles matching current monitors. |

## Mocked or isolated abstractions

Visible tests focus on service and model logic that can be run without real monitor hardware. The codebase has interfaces such as `IMonitorController`, `IDdcController`, `IBasicMonitorController`, and `IMonitorData`, but the inspected test names do not show broad integration tests that mock every OS API boundary.

The most OS-dependent pieces are not obviously exercised end-to-end in unit tests:

- `DisplayConfigInventory` calling DisplayConfig APIs.
- `WmiController` querying and invoking `root\WMI` classes.
- `DdcCiController` using logical monitor handles, physical monitor handles, capabilities strings, and VCP reads/writes.
- WinUI slider interactions calling into `MonitorManager`.

## Behaviours covered

The visible tests appear to cover:

- Monitor identity normalization and comparison.
- Legacy ID migration and settings/profile cleanup.
- Capability parsing and VCP value handling.
- Linked brightness planning and settings persistence.
- Crash detection/recovery around DDC discovery.
- Blacklist matching.
- Monitor settings rebuild/retention behavior.

## Coverage gaps relevant to VDI and virtual displays

The visible tests do not appear to cover these VDI-focused conditions directly:

- Virtual displays or remote/VDI sessions.
- DisplayConfig returning active paths that do not map to WMI brightness or DDC/CI capabilities.
- WMI brightness classes absent, inaccessible, or returning partial data in a remote session.
- `EnumDisplayMonitors` returning logical handles for which physical-monitor retrieval fails.
- DDC/CI capability strings absent/unparseable in a virtual display stack, outside parser unit tests.
- A display identity mismatch between WMI `InstanceName` and DisplayConfig `DevicePath` in live discovery.
- UI distinction between no displays, no brightness-capable displays, and write failure.
- Permission/elevation matrix for WMI brightness methods.
- Multi-monitor VDI scenarios where display numbers, device paths, and GDI names may change between sessions.

## Likely run approach

For this documentation-only change, no tests were changed. If a future code investigation adds tests, start with `src/modules/powerdisplay/PowerDisplay.Lib.UnitTests/PowerDisplay.Lib.UnitTests.csproj`, build it with the repo build scripts on Windows, and run through Visual Studio Test Explorer or `vstest.console.exe` per repository guidance.
