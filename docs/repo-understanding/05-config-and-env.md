# Configuration and Environment

## Settings File Location and Format

PowerToys stores all user configuration in a single JSON file:

```
%LocalAppData%\Microsoft\PowerToys\settings.json
```

Example structure:
```json
{
  "version": "2.0",
  "enabled": true,
  "modules": {
    "ColorPicker": {
      "enabled": true,
      "properties": {
        "activation_shortcut": "Win+Shift+C",
        "auto_copy_color": true,
        "color_format": "HEX"
      }
    },
    "FancyZones": {
      "enabled": false,
      "properties": {
        "activation_shortcut": "Win+\`",
        "move_default_enabled": true
      }
    }
  },
  "general": {
    "startup": true,
    "run_elevated": false,
    "theme": "dark"
  }
}
```

### Settings File Characteristics

- **Format**: JSON with nested structure
- **Scope**: Global (applies to all users) or per-user
- **Persistence**: Atomic writes to prevent corruption
- **Versioning**: "version" field allows migration logic for format upgrades
- **Access**: Readable by modules and Settings UI through `SettingsAPI` helpers
- **Backup**: No automatic backup; users may keep manual copies

### Key Setting Categories

| Setting | Scope | Example Values | Purpose |
|---------|-------|---------------|---------| 
| `enabled` (module) | Per-module | true/false | Whether module is active |
| `activation_shortcut` | Per-module | "Win+Shift+C" | Hotkey for module |
| `startup` (general) | Global | true/false | Run PowerToys on Windows startup |
| `run_elevated` | Global | true/false | Run with admin privileges |
| `theme` | Global | "light"/"dark"/"auto" | UI theme preference |

---

## Environment Variables

PowerToys does not heavily rely on environment variables, but some may affect behavior:

| Variable | Purpose | Default | Notes |
|----------|---------|---------|-------|
| `POWERTOYS_LOG_LEVEL` | Logging verbosity | INFO | Can be set to DEBUG for verbose logs |
| `POWERTOYS_DATA_DIR` | Custom data directory | `%LocalAppData%\Microsoft\PowerToys` | May be set in enterprise scenarios |
| `POWERTOYS_INSTALLER_SCOPE` | Installer scope | per-user | "per-user" or "machine" |

Most environment variables are read during startup and cached; runtime changes do not take effect until restart.

---

## Registry Keys

PowerToys uses the Windows Registry for several purposes:

### HKEY_CURRENT_USER (Per-User)

```
HKEY_CURRENT_USER\Software\Microsoft\PowerToys\
  - General\ (global settings)
  - Modules\{ModuleKey}\  (per-module settings, legacy)
  - Update\ (update check results)
```

### HKEY_LOCAL_MACHINE (Machine-Wide, Enterprise GPO)

```
HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\PowerToys\
  - Enabled (GPO: enable/disable PowerToys)
  - {ModuleKey}\Enabled (GPO: enable/disable specific module)
  - {ModuleKey}\Restricted (GPO: restrict user changes to module)
```

### Installer/Shortcuts

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
  - PowerToys = "C:\Program Files\PowerToys\PowerToys.exe"
```

**Note**: Most modern settings are in `settings.json`; registry keys are mainly for legacy support and enterprise policies (GPO).

---

## Command-Line Arguments

PowerToys.exe accepts several command-line arguments:

| Argument | Effect | Example |
|----------|--------|---------|
| `--elevated` | Run with admin privileges | `PowerToys.exe --elevated` |
| `--dont-show-again` | Skip welcome/tutorial on startup | `PowerToys.exe --dont-show-again` |
| `--version` | Print version and exit | `PowerToys.exe --version` |
| (no args) | Run normally | `PowerToys.exe` |

These are parsed in `src/runner/main.cpp` early in startup. Most users run without arguments (via shortcut or auto-start).

---

## Logging Configuration

PowerToys logging is controlled by `src/common/logger/`:

### Log File Location

```
%LocalAppData%\Microsoft\PowerToys\PowerToys.log
```

### Log Levels

- **ERROR**: Critical failures (module load errors, crash info)
- **WARN**: Recoverable issues (hotkey conflict, GPO enforcement)
- **INFO**: General operation (module enabled, hotkey pressed)
- **DEBUG**: Verbose details (function entry/exit, state changes)

Default level is **INFO**; can be changed via `POWERTOYS_LOG_LEVEL` environment variable or registry.

### Log Format

```
[2024-01-15 14:23:45.123] [PowerToys] [INFO] Color Picker enabled
[2024-01-15 14:23:46.456] [PowerToys] [ERROR] Failed to load module colorPicker: DLL not found
```

Each log line includes timestamp, component name, level, and message.

---

## Telemetry Configuration

PowerToys telemetry is opt-in and privacy-respecting (no PII collected).

### Telemetry Events

Logged when:
- Module is enabled/disabled
- Hotkey is pressed
- Crash or unhandled exception occurs
- Feature is used (e.g., Color Picker color copied)

### Privacy Model

- **Local storage**: Events stored in local ETW trace buffer
- **Optional upload**: User can opt-in to send events to Microsoft telemetry service
- **No PII**: File names, clipboard contents, or user data are never collected
- **Enterprise**: IT admins can disable telemetry via GPO

### Telemetry Data Access

Users can inspect local telemetry events using Windows Event Viewer:
```
Event Viewer > Applications and Services Logs > PowerToys
```

---

## Developer / Test Mode

### Debug Build

When built in Debug configuration:
- All modules load from build output directory
- Logging set to DEBUG level
- PDB symbol files available for debugger
- Assertions are checked at runtime

### Release Build

- Optimized for performance and size
- Logging set to INFO level
- All symbols stripped from release binaries
- Modules load from installation directory

### Test Mode

Unit tests typically:
- Isolate settings file to temp directory
- Mock keyboard hook (to avoid interfering with real hotkeys)
- Create module instances directly without going through DLL loading
- May disable telemetry

---

## Installation Scope

PowerToys can be installed in two scopes:

### Per-User (Recommended)

```
Installation directory: %LocalAppData%\Microsoft\PowerToys\ or C:\Users\{Username}\AppData\Local\Microsoft\PowerToys\
```

- No admin privileges required to install or uninstall
- Settings stored in user's AppData
- Multiple users can have different settings
- Startup shortcut in user's Startup folder

### Machine-Wide

```
Installation directory: C:\Program Files\PowerToys\ or C:\Program Files (x86)\PowerToys\
```

- Requires admin privileges to install or uninstall
- Settings still stored per-user in AppData
- All users on machine have access to the same executable
- Startup controlled via machine-wide registry key or Group Policy

## Group Policy (GPO) Integration

Enterprise environments can control PowerToys behavior via Active Directory policies.

### Policy Settings

```
Computer Configuration > Administrative Templates > Microsoft > PowerToys
  - Enable/disable PowerToys globally
  - Enable/disable specific modules
  - Restrict user changes to certain settings (read-only)
  - Enforce specific hotkey assignments
```

### Policy Enforcement

- Policies read from `HKEY_LOCAL_MACHINE\Software\Policies\Microsoft\PowerToys\`
- Checked at startup and periodically during runtime
- If user tries to change a restricted setting, change is rejected and warning shown
- IT admins have full control; users cannot override policies

---

## Local vs Dev vs Test vs Prod

### Local (Developer Machine)

- Build from source using `tools/build/build.ps1`
- Settings and logs in local AppData
- Can attach debugger
- No installer; modules run from build output directories

### Dev (Development Branch)

- Latest features (may be unstable)
- Nightly builds available
- Settings synced across updates
- Telemetry enabled by default (opt-out available)

### Test

- Test-specific isolation:
  - Settings file mocked or isolated to temp directory
  - Hotkey registration skipped or mocked
  - Telemetry disabled
  - IPC may be mocked for unit tests
- Unit tests run with GoogleTest (C++) or XUnit (C#)

### Prod (Release)

- Stable, tested build
- Available on Microsoft Store, WinGet, direct download
- Settings persist across updates
- Telemetry enabled (opt-in)
- Updates are automatic via Microsoft Store or WinGet

---

## Secrets and Credentials

PowerToys **does not store secrets or credentials** in settings.json or registry. The system is designed to avoid requiring authentication.

However, some modules may interact with:
- **Clipboard** (Advanced Paste, Color Picker, etc.)
- **File system** (PowerRename, File Locksmith)
- **Registry** (Registry Preview, Environment Manager)

**No API keys, passwords, or tokens are stored by PowerToys itself.** If a module needs external authentication (e.g., for future cloud features), credentials would be managed by the host OS (Credential Manager) or an external service, not by PowerToys.

---

## Summary

PowerToys configuration is minimal and user-friendly:

- **Primary storage**: JSON file (`settings.json`)
- **Enterprise control**: Group Policy / Registry
- **Environment variables**: Few; mainly for logging level
- **Logging**: Local file in AppData; can be verbose or silent
- **Telemetry**: Opt-in; local collection before optional upload
- **No secrets**: System avoids credential storage

Configuration changes take effect immediately for Settings UI changes, or at next hotkey press for module behavior changes. Global settings (startup behavior) take effect at next PowerToys restart.
