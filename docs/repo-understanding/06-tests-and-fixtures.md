# Tests and Fixtures

## Test Framework and Structure

PowerToys uses multiple testing frameworks depending on the component:

| Framework | Language | Used For | Location |
|-----------|----------|----------|----------|
| GoogleTest (gtest) | C++ | Native module unit tests | `src/modules/*/unittests/` |
| XUnit | C# | .NET module and Settings UI tests | `src/modules/*/tests/`, `src/settings-ui/*/UnitTests/` |
| MSTest | C# | Legacy tests (being migrated to XUnit) | Various |
| WinAppDriver | UI Automation | End-to-end Settings UI tests | `src/settings-ui/UITest-Settings/` |

## Running Tests

### Build Tests

Before running tests, build the test project first:

```powershell
# From repository root
.\tools\build\build.ps1 -Platform x64 -Configuration Debug -Path "src/modules/colorPicker/unittests"
```

### Run C++ Unit Tests

Via Visual Studio Test Explorer:
```
Test > Test Explorer (Ctrl+E, T) > Run Tests
```

Or via command line:
```powershell
.\tools\build\build.ps1 -Platform x64 -Configuration Debug -Path "src/modules/colorPicker/unittests"
cd "src/modules/colorPicker/unittests"
dotnet test  # or vstest.console.exe
```

### Run C# Unit Tests

Via Visual Studio Test Explorer or:
```powershell
cd "src/settings-ui/Settings.UI.UnitTests"
dotnet test
```

### Run UI Tests

Requires WinAppDriver installed and WinUI/Settings UI running:
```powershell
# Install WinAppDriver first (see doc/devdocs/development/ui-tests.md)
cd "src/settings-ui/UITest-Settings"
dotnet test
```

## Test Organization by Component

### Runner Tests

**Location**: `src/common/UnitTests-CommonLib/` (general Runner utilities)

**What's tested**:
- Logger initialization
- Settings file I/O
- String utilities
- Hotkey conflict detection
- IPC message parsing

**Example test**: Verify hotkey conflict detector correctly identifies conflicts with system hotkeys.

### Module Interface Tests

**Location**: None specific (each module tests its own interface implementation)

**What's tested**:
- Module enable/disable lifecycle
- Config serialization/deserialization
- Hotkey registration
- Module initialization

**Example test**: `colorPicker/unittests/` tests that ColorPickerModule correctly registers its hotkey and responds to `on_hotkey()`.

### Settings UI Tests

**Location**: 
- `src/settings-ui/Settings.UI.UnitTests/` – ViewModel and control tests
- `src/settings-ui/UITest-Settings/` – End-to-end Settings UI tests

**What's tested**:
- ViewModel binding and state changes
- UI control rendering and interaction
- Settings persistence through IPC
- Module list enumeration

**Example test**: Verify toggling a module's enable switch triggers IPC message send and state updates.

### Module-Specific Tests

Each major module has its own test project:

| Module | Test Project | Test Type | Key Tests |
|--------|--------------|-----------|-----------|
| ColorPicker | `src/modules/colorPicker/unittests/` | C++ unit | Color format conversion, hotkey handling |
| PowerRename | `src/modules/powerrename/unittests/` | C++ unit | Regex pattern matching, rename operation |
| FancyZones | `src/modules/fancyzones/*/tests/` | C# unit | Zone layout calculation, window snap |
| ImageResizer | `src/modules/imageresizer/tests/` | C# unit | Image resize operations, format preservation |
| AdvancedPaste | `src/modules/AdvancedPaste/*Tests/` | Mixed | Paste formatting, JSON parsing |

### Common Library Tests

**Location**: `src/common/UnitTests-*`

**What's tested**:
- JSON parsing (`src/common/utils/json.h`)
- IPC message framing (`src/common/interop/two_way_pipe_message_ipc.h`)
- String utilities
- Path manipulation
- Registry access wrappers

## Fixtures and Test Data

### Settings File Fixtures

Test projects often include sample `settings.json` fixtures for testing settings loading:

```json
{
  "version": "2.0",
  "modules": {
    "ColorPicker": {
      "enabled": true,
      "properties": {
        "activation_shortcut": "Win+Shift+C"
      }
    }
  }
}
```

Located in test project directories (e.g., `src/settings-ui/Settings.UI.UnitTests/TestData/`).

### Mock Data

Tests use mock objects for:
- **IPC Messages**: JSON strings representing Settings UI commands
- **Module Metadata**: Mock module names, keys, and hotkeys
- **Keyboard Events**: Simulated keystroke data
- **Registry**: Mock registry queries for GPO testing

### Sample File Fixtures

Some tests use sample files (located in `testdata/` subdirectories):

| Module | Fixtures | Purpose |
|--------|----------|---------|
| PowerRename | Sample files with various names | Test batch rename patterns |
| ImageResizer | Sample images (PNG, JPG, TIFF) | Test image resizing |
| Preview Pane | Sample documents (PDF, etc.) | Test preview rendering |

## Coverage and Gaps

### Well-Tested Areas

- Module enable/disable lifecycle
- Settings parsing and persistence
- Hotkey registration
- IPC message framing
- Regex and text processing

### Known Gaps

- End-to-end integration tests (mostly covered by manual testing)
- Real keyboard hook behavior (tested in isolation, not globally)
- Multi-monitor edge cases (tested per-monitor, limited combo testing)
- Performance tests (not comprehensive)
- Crash and memory leak detection (limited coverage)

## Test Execution During Development

### Local Development Workflow

1. Make code changes
2. Build changed project with `tools/build/build.ps1`
3. Run unit tests via Test Explorer or command line
4. Verify tests pass before committing

### CI/CD Pipeline

Tests run automatically on:
- Every pull request (verified before merge)
- Nightly builds (comprehensive suite)
- Release builds (full test suite + integration tests)

GitHub Actions workflow files in `.github/workflows/` orchestrate test runs.

## Adding Tests

### For a New Module

1. Create test project next to module: `src/modules/mymodule/unittests/`
2. Reference GoogleTest for C++ or XUnit for C#
3. Write tests covering:
   - Module initialization (`powertoy_create()`)
   - Enable/disable lifecycle
   - Hotkey registration
   - Config serialization
   - Core functionality

### For Existing Modules

Add test cases to existing test projects:

```cpp
// Example C++ test in src/modules/colorPicker/unittests/
TEST(ColorPickerModule, EnableSetsUpHotkey) {
    auto module = /* create module */;
    module->enable();
    auto hotkeys = module->get_hotkeys(nullptr, 0);
    ASSERT_GT(hotkeys, 0);  // Should register at least one hotkey
}
```

## Documentation for Tests

### Build Guidelines

- `tools/build/BUILD-GUIDELINES.md` – Build system and exit codes

### Development Guidelines

- `doc/devdocs/development/guidelines.md` – Testing conventions and best practices

### UI Tests

- `doc/devdocs/development/ui-tests.md` – WinAppDriver setup and usage

### Fuzzing

- `doc/devdocs/tools/fuzzingtesting.md` – Fuzzing tests for robust input handling

## Test Discipline Rules

Per `CONTRIBUTING.md` and development guidelines:

1. **Add tests when changing behavior** – All feature additions or bug fixes should include tests
2. **Avoid breaking existing tests** – Do not remove or skip tests without justification
3. **Run tests before committing** – Local test runs verify changes don't break existing functionality
4. **High-risk areas need tests** – File I/O, IPC, registry access, and keyboard hooks must be thoroughly tested

## Quick Reference: Running Tests

| Scenario | Command |
|----------|---------|
| Build and test a module | `.\tools\build\build.ps1 -Path "src/modules/colorPicker/unittests"` then open Test Explorer |
| Run Settings UI tests | `cd src/settings-ui/Settings.UI.UnitTests && dotnet test` |
| Run common lib tests | `cd src/common/UnitTests-CommonLib && dotnet test` |
| Run all tests (CI) | GitHub Actions runs automatically on PR |

## Summary

PowerToys testing is multi-layered:
- **Unit tests** cover individual components (modules, utilities, settings)
- **Integration tests** (limited) verify component interactions
- **UI tests** (limited) verify Settings UI functionality
- **Fixtures** provide sample data and mock objects
- **CI/CD** enforces test passage before merge

Adding or modifying tests is essential when changing behavior; skipping tests is only acceptable for documentation-only or UI-only changes.
