# Vulnerability Review

## Review metadata

- Repository: PowerToys
- Commit: 19c2c28dd9f6eff9097df662cc854141d0fc9bc0
- Branch: work
- Reviewed at: 2026-07-21T10:46:47+00:00
- Review type: Bounded surface application-security review
- Languages/frameworks: C++, C#, WinUI/WPF, WebView2/Monaco, Windows named pipes/COM/registry shell extensions, WiX installer projects, PowerShell build/settings scripts.
- Areas prioritised: runner/settings IPC and elevation controls, Advanced Paste process/named-pipe and AI/clipboard boundaries, Registry Preview file/WebView2/registry handoff paths, shared named-pipe IPC implementation, updater/launcher subprocess surfaces.
- Areas not reviewed or only sampled: most individual PowerToys modules, most UI-only settings pages, installer custom actions beyond high-risk search hits, CI/CD workflows, generated resources, test fixtures, docs website, dependency provenance and transitive dependency CVE reachability.

## Executive summary

- Validated findings: 0 Critical, 0 High, 0 Medium.
- Prompt human action required: none from this bounded review.
- Strongest overall risk theme reviewed: local process and user-boundary IPC/elevation surfaces, especially named pipes used between the runner, Settings UI, Quick Access, and Advanced Paste.
- Important limitations: this was a bounded static source/configuration review on a large Windows desktop monorepo. It did not build or execute PowerToys, did not run a full dependency audit, and did not review every module line by line.

No validated Medium, High, or Critical vulnerability was identified within the review scope.

## Validated findings

| ID | Severity | Confidence | Status | Title | Primary location | Impact |
|---|---|---|---|---|---|---|

## Reviewed attack surfaces

- **PowerToys runner and Settings UI IPC/elevation path.** Reviewed the runner's Settings launch and IPC setup, including creation of `TwoWayPipeMessageIPC` with the current process token in `src/runner/settings_window.cpp` and Settings-side startup/pipe wiring in `src/settings-ui/Settings.UI/SettingsXAML/App.xaml.cs`. The runner constructs the Settings IPC object and starts it with a token (`current_settings_ipc->start(hToken)`), while Settings consumes pipe names from command-line arguments passed by the runner.
- **Shared two-way named-pipe implementation.** Reviewed `src/common/interop/two_way_pipe_message_ipc.cpp`, especially pipe creation, per-instance connection handling, and the optional token-based security adjustment path. This is a key trust boundary because Settings and other UI processes can send messages that affect runner behavior.
- **Advanced Paste process boundary.** Reviewed C++ process launch and pipe setup in `src/modules/AdvancedPaste/AdvancedPasteModuleInterface/AdvancedPasteProcessManager.cpp`, the C# named-pipe client in `src/modules/AdvancedPaste/AdvancedPaste/Helpers/NamedPipeProcessor.cs`, and message dispatch in `src/modules/AdvancedPaste/AdvancedPaste/AdvancedPasteXAML/App.xaml.cs`. The reviewed attack surface includes local command-line/pipe-name exposure, clipboard-triggered operations, and AI-service paths using locally configured credentials.
- **Registry Preview file/WebView2/registry handoff.** Reviewed registry-file loading and size clamping, WebView2/Monaco initialization and script messaging, URI-launch prompts, and the handoff to `regedit.exe` in `src/modules/registrypreview/RegistryPreviewUILib/RegistryPreviewMainPage.Utilities.cs` and `src/modules/registrypreview/RegistryPreviewUILib/Controls/MonacoEditor/MonacoEditorControl.xaml.cs`.
- **High-risk sink search across sampled surfaces.** Searched for subprocess execution, shell execution, named pipes, dynamic script execution, WebView2 host object exposure, deserialization, file reads/writes, registry operations, and network/API credential usage under the prioritized runner, settings, common IPC, Advanced Paste, Registry Preview, launcher, updater, and installer areas.

## Non-findings and dismissed candidates

- **Settings IPC spoofing/elevation candidate dismissed.** The Settings/runner IPC path initially looked sensitive because Settings can send action messages such as restart/elevation requests. The shared pipe implementation creates named pipes in `TwoWayPipeMessageIPC::TwoWayPipeMessageIPCImpl::start_named_pipe_server`, and the runner starts the Settings IPC with the current process token so the implementation can adjust pipe security for that token. The reviewed evidence supports a local same-user IPC design rather than a validated cross-user or unelevated-to-elevated privilege-escalation path in this bounded pass.
- **Advanced Paste pipe hijack candidate dismissed below reporting bar.** The runner creates a GUID-suffixed pipe name and passes it to the Advanced Paste child process; the pipe is outbound-only from the runner and the C# client only reads line-delimited runner messages. A same-user local process that learned the transient pipe name might race the legitimate child and receive UI/hotkey control messages, but the reviewed path did not establish command execution, credential disclosure, clipboard disclosure, or an unauthorized write/control primitive with Medium-or-higher impact.
- **Registry Preview WebView2 script injection candidate dismissed.** Registry file content is passed into Monaco via `ExecuteScriptAsync`, but the reviewed setter encodes file text with `HttpUtility.JavaScriptStringEncode` before inserting it into a JavaScript string. Host objects are disabled, default dialogs/context menus and password/autofill are disabled, and external URI launches require a user-initiated new-window event plus a confirmation dialog. The reviewed controls prevented a validated file-content-to-code-execution or sensitive-data disclosure path.
- **Registry merge via `regedit.exe` candidate dismissed.** Registry Preview can pass a selected/current `.reg` file to `regedit.exe`, but this is an explicit local user action that invokes Windows Registry Editor and UAC behavior. The reviewed code quotes the filename argument and first checks that the file exists, so no validated attacker-controlled argument injection or silent privilege crossing was established.
- **Advanced Paste AI/clipboard exfiltration candidate dismissed.** Advanced Paste can send user clipboard content to configured AI services for explicit AI transforms, but the reviewed paths require the feature and credentials to be configured and are driven by local user hotkeys/actions. No hidden or unauthenticated remote trigger that forces arbitrary clipboard exfiltration was established in the sampled code.

## Limitations

- This was a bounded source and configuration review, not a penetration test.
- The full repository was not reviewed line by line.
- No dependency installation, build, migrations, package hooks, or application startup commands were performed.
- Most dependency provenance and transitive dependency risk was outside scope.
- Absence of a reported finding is not proof that the repository is secure.
- Runtime configuration, infrastructure, secrets, installed application paths, Windows ACL defaults, enterprise policy, and external services not represented in the repository may change the risk.
