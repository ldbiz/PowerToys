# Caveats and Unknowns

This document captures uncertainty and gaps in the understanding of the PowerToys repository. These points should be clarified through code inspection, testing, or discussion with maintainers.

## Architectural Questions

### Module Dependency Order

**Question**: Are there dependencies between modules during loading? Can one module rely on another being loaded first?

**Current understanding**: Module loading appears to be order-independent based on the code structure, but the list of modules is hardcoded in `main.cpp`. If modules do have dependencies, the order matters.

**Why uncertain**: The module loading logic doesn't explicitly check for dependencies or failures in dependent modules.

**How to verify**: Trace through `powertoy_module.cpp` to see if load failures are handled gracefully; check if any modules import symbols from other modules.

---

### IPC Message Versioning

**Question**: How does the IPC protocol handle backward compatibility when the message format changes?

**Current understanding**: The settings file has a "version" field for schema migration, but it's unclear if IPC messages have explicit versioning.

**Why uncertain**: The IPC code in `two_way_pipe_message_ipc.h` processes JSON without an explicit version check in the visible code.

**How to verify**: Review Settings UI version handling and Runner's IPC message parser.

---

### Hotkey Storage and Synchronization

**Question**: Are hotkey assignments stored separately from module settings, or are they part of the module's config?

**Current understanding**: Hotkeys appear to be stored in the module's config JSON, but the settings UI may have a separate hotkey control that binds to the module config.

**Why uncertain**: Both `get_hotkeys()` method and `set_config()` deal with hotkeys; unclear which is source of truth.

**How to verify**: Check Color Picker module's hotkey storage in `settings.json`; trace how Settings UI updates a hotkey.

---

## Performance and Scalability Questions

### Keyboard Hook Performance Under Load

**Question**: How does the keyboard hook perform with many modules loaded and many hotkeys registered?

**Current understanding**: The keyboard hook runs at low-level OS priority; callback must be very fast. The code suggests conflict detection and matching is O(N) in the number of modules/hotkeys.

**Why uncertain**: No visible performance tests or benchmarks for hotkey matching with 30+ modules.

**How to verify**: Profile the hook with multiple modules enabled; measure latency on keypresses.

---

### Settings File Polling and Reloading

**Question**: Does the Runner watch `settings.json` for external changes (e.g., via GPO or IT admin scripts)?

**Current understanding**: Settings are read once at startup and updated via IPC from Settings UI. External file modifications may not be detected.

**Why uncertain**: No visible file watcher or polling mechanism in the code.

**How to verify**: Change `settings.json` externally (edit file with text editor) and check if Runner picks up changes; restart Runner and verify settings are reloaded.

---

## Behavioral Uncertainties

### Module Disable Cleanup

**Question**: What happens if `disable()` on a module fails or doesn't fully clean up? Is there a timeout or forced cleanup?

**Current understanding**: `disable()` is called and expected to clean up; no visible error handling or timeout.

**Why uncertain**: Modules may hang or leak resources if disable fails.

**How to verify**: Test a module that has a long-running thread; call disable and see if it blocks the Runner.

---

### Hotkey Conflict When System Hotkey Changes

**Question**: If the system hotkey assignment changes (e.g., Windows adds a new hotkey in an OS update), how does PowerToys detect and handle it?

**Current understanding**: Conflict detection happens at hotkey assignment time in Settings UI, not continuously.

**Why uncertain**: No visible runtime conflict detection or re-validation.

**How to verify**: Test with an OS update that introduces new hotkeys; check if PowerToys detects conflicts.

---

### Settings UI Process Lifetime

**Question**: Is Settings UI a single process that stays alive, or does it launch/exit on demand?

**Current understanding**: Settings UI appears to launch when user clicks tray icon and may exit when closed; it's not persistent.

**Why uncertain**: Settings UI's process management isn't clearly documented in the code.

**How to verify**: Monitor processes with Task Manager; click tray icon and observe Settings UI lifetime.

---

### IPC Named Pipe Reconnection

**Question**: If Settings UI crashes or IPC pipe breaks, can the two processes reconnect without restarting PowerToys?

**Current understanding**: Named pipe is established on first connection; unclear what happens if pipe breaks.

**Why uncertain**: Error handling and reconnection logic in `two_way_pipe_message_ipc.h` is not fully visible in brief review.

**How to verify**: Kill Settings UI process and try to open it again; check if IPC reconnection works.

---

## External System Integration Questions

### GPO Synchronization Timing

**Question**: Are GPO policy changes detected in real-time, or only at startup/login?

**Current understanding**: GPO keys are likely read at startup and periodically (possibly every login).

**Why uncertain**: The polling/refresh interval for GPO checks is not documented.

**How to verify**: Change a GPO setting and observe when PowerToys applies it (requires test domain setup).

---

### Telemetry Network Behavior

**Question**: Does telemetry collection block the Runner's message loop, or is it async?

**Current understanding**: ETW tracing is typically async; telemetry should not block.

**Why uncertain**: The telemetry subsystem's threading model is not fully reviewed.

**How to verify**: Profile telemetry calls during hotkey dispatch; measure latency impact.

---

### Module DLL Unloading

**Question**: When a module is disabled, is its DLL unloaded from memory, or kept loaded but inactive?

**Current understanding**: DLLs are loaded once at startup and likely kept in memory; disable doesn't unload.

**Why uncertain**: `FreeLibrary()` is not visible in the module loading code path.

**How to verify**: Monitor memory usage after disabling a module; check if memory is reclaimed.

---

## Testing and Coverage Gaps

### End-to-End Integration Tests

**Question**: Are there end-to-end tests that simulate the full startup → hotkey → action → settings UI flow?

**Current understanding**: Unit tests exist; integration tests appear limited. CI likely has some integration tests not visible in the repo.

**Why uncertain**: Repository only shows unit test projects; full integration test suite may be in CI pipeline.

**How to verify**: Check `.github/workflows/` for full CI test suite; look for integration test jobs.

---

### Module Isolation Testing

**Question**: Are modules tested in isolation (mocked Runner), or integrated with the real Runner during testing?

**Current understanding**: Each module likely has unit tests; unclear if integration tests run modules with real Runner.

**Why uncertain**: Test project structures are not fully examined.

**How to verify**: Review test project configurations and dependencies.

---

### Multi-Module Hotkey Scenarios

**Question**: Are there tests for hotkey conflicts, two modules with same hotkey, or rapid repeated hotkeys?

**Current understanding**: Conflict detection exists; unclear if edge cases are tested.

**Why uncertain**: Test coverage for hotkey scenarios is not comprehensive.

**How to verify**: Search test projects for hotkey conflict and edge-case tests.

---

## Documentation and Clarity Gaps

### Module Template Documentation

**Question**: Is the module template (`tools/project_template/ModuleTemplate`) kept up-to-date with current best practices?

**Current understanding**: Template exists; unclear if it reflects current architectural patterns.

**Why uncertain**: Template code is not recently reviewed.

**How to verify**: Compare template with recent module implementations (e.g., Advanced Paste).

---

### Settings Schema Documentation

**Question**: Is there a formal schema or documentation for the `settings.json` structure?

**Current understanding**: Schema is implicit in the code; no formal JSON Schema or documentation found.

**Why uncertain**: Settings structure is inferred from code; no `settings.schema.json` file visible.

**How to verify**: Search for settings documentation or schema files; check if one should be created.

---

### IPC Message Reference

**Question**: Is there a documented list of all IPC messages and their JSON schemas?

**Current understanding**: IPC messages are JSON; no formal specification found.

**Why uncertain**: Message formats are embedded in code; no centralized documentation.

**How to verify**: Grep for JSON messages in code and compile a list with schemas.

---

## Known Limitations and Design Constraints

### No Real-Time Module Updates

**Limitation**: Modules are loaded at startup; adding/removing modules requires PowerToys restart.

**Why**: DLL loading is not dynamic; no hot-loading mechanism.

---

### Single PowerToys Instance

**Limitation**: Only one PowerToys process can run at a time (enforced by mutex).

**Why**: Simplifies state management and prevents IPC conflicts.

---

### No Module-to-Module Communication

**Limitation**: Modules cannot directly communicate with each other; all communication flows through Runner.

**Why**: Maintains loose coupling and isolation; simplifies lifecycle management.

---

### Settings Persistence Timing

**Limitation**: Settings are written to disk asynchronously or not immediately on every change; there may be a small window where changes are in memory but not persisted.

**Why**: Batch writes for performance; IPC messages are processed quickly.

---

## Recommended Next Steps for Clarification

1. **Profile hotkey dispatch** – Measure actual latency to understand performance constraints
2. **Trace IPC protocol** – Capture network/pipe traffic to document message formats
3. **Review CI/CD** – Check GitHub Actions workflows for integration tests and performance benchmarks
4. **Interview maintainers** – Clarify architectural decisions and known limitations
5. **Build and test locally** – Verify assumptions about process lifetime, IPC reconnection, and module loading
6. **Create formal specifications** – Document settings schema, IPC protocol, and module interface formally

## Summary

The main uncertainties in PowerToys are around:
- **Performance** – No visible benchmarks or load testing
- **Error handling** – Unclear how failures are handled (e.g., module disable failure)
- **External changes** – How system GPO, registry, or file changes are detected
- **Integration** – Full end-to-end test coverage is unclear
- **Documentation** – Formal specs for settings, IPC, and module loading are missing

These gaps don't indicate bugs or missing functionality; they reflect areas where the codebase could be more explicitly documented or tested. Most behavior is likely correct but implicit in the implementation.
