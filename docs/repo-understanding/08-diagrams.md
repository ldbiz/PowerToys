# Diagrams

## 1. Runtime Flow Diagram

Execution flow from startup to hotkey dispatch:

```mermaid
graph TD
    A["PowerToys.exe starts"]
    B["Initialize logger & mutex"]
    C["Create system tray icon"]
    D["Install global keyboard hook"]
    E["Load module DLLs"]
    F["For each enabled module: call enable()"]
    G["Register hotkeys from modules"]
    H["Enter Windows message loop"]
    I{"Message received?"}
    J{"Hotkey pressed?"}
    K{"Tray icon clicked?"}
    L["Call module.on_hotkey()"]
    M["Module launches UI or performs action"]
    N["Send IPC msg to Settings UI"]
    O["Settings UI opens or updates"]
    P["Return to message loop"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I -->|Yes| J
    I -->|Timeout| P
    J -->|Yes| L
    J -->|No| K
    K -->|Yes| N
    K -->|No| P
    L --> M
    M --> P
    N --> O
    O --> P
    P --> I
```

---

## 2. Module Loading and Initialization

Detailed sequence of module DLL loading and enablement:

```mermaid
sequenceDiagram
    participant Runner as PowerToys Runner
    participant DLL as Module DLL
    participant Module as Module Object
    participant Hook as Keyboard Hook

    Runner->>DLL: LoadLibrary()
    DLL-->>Runner: DLL loaded
    Runner->>DLL: powertoy_create()
    DLL->>Module: new PowerToyImpl()
    Module-->>DLL: Module instance
    DLL-->>Runner: Module*

    Runner->>Module: get_key()
    Module-->>Runner: "colorPicker"

    Runner->>Module: get_config()
    Module-->>Runner: JSON config

    alt If enabled in settings
        Runner->>Module: enable()
        Module->>Hook: Register hotkey
        Hook-->>Module: Hotkey registered
        Module-->>Runner: Success
        
        Runner->>Module: get_hotkeys()
        Module-->>Runner: Hotkey[]
        
        Runner->>Hook: Add hotkeys to global hook
    end
```

---

## 3. Hotkey Press and Dispatch

Low-level flow when user presses a hotkey:

```mermaid
graph LR
    A["User presses<br/>hotkey"] --> B["OS calls<br/>keyboard hook"]
    B --> C["centralized_kb_hook.cpp<br/>callback invoked"]
    C --> D["Translate to<br/>Hotkey struct"]
    D --> E["Match against<br/>registered hotkeys"]
    E --> F{"Match<br/>found?"}
    F -->|Yes| G["Identify module"]
    F -->|No| H["Pass to next hook<br/>in chain"]
    G --> I["Call<br/>module.on_hotkey()"]
    I --> J["Module performs<br/>action or launches<br/>external app"]
    J --> K["Return to Runner<br/>message loop"]
    H --> K
```

---

## 4. Settings UI and IPC Communication

Data flow between Settings UI and Runner:

```mermaid
graph TD
    A["User opens Settings UI"]
    B["Settings UI connects<br/>to named pipe"]
    C["Runner listens on<br/>named pipe"]
    D["User changes setting"]
    E["Settings UI sends<br/>JSON message"]
    F["Runner receives<br/>IPC message"]
    G["Runner updates<br/>settings.json"]
    H["Runner calls<br/>module.set_config()"]
    I["Module applies<br/>new config"]
    J["Settings persisted<br/>to disk"]
    K["Next run loads<br/>new settings"]

    A --> B
    B --> C
    D --> E
    E --> F
    F --> G
    F --> H
    H --> I
    G --> J
    J --> K
```

---

## 5. Module Architecture and Component Interaction

High-level view of module types and how they fit together:

```mermaid
graph TB
    subgraph Runner["PowerToys Runner (main process)"]
        TrayIcon["Tray Icon Handler"]
        ModuleLoader["Module Loader"]
        KbHook["Keyboard Hook"]
        IPCServer["IPC Named Pipe Server"]
    end

    subgraph SettingsUI["Settings UI (separate process)"]
        MainWindow["Main Window XAML"]
        IPCClient["IPC Named Pipe Client"]
    end

    subgraph Modules["Module DLLs (loaded by Runner)"]
        SimpleModule["Simple Module<br/>(embedded UI)"]
        ExternalModule["External Launcher Module<br/>(launches separate app)"]
        ShellExtModule["Shell Extension Module<br/>(File Explorer context menu)"]
    end

    subgraph External["External Applications"]
        ColorPickerUI["Color Picker UI"]
        ImageResizerUI["Image Resizer UI"]
    end

    ModuleLoader -->|Loads| SimpleModule
    ModuleLoader -->|Loads| ExternalModule
    ModuleLoader -->|Loads| ShellExtModule
    
    KbHook -->|Calls on_hotkey| SimpleModule
    KbHook -->|Calls on_hotkey| ExternalModule
    
    ExternalModule -->|Launches| ColorPickerUI
    ExternalModule -->|Launches| ImageResizerUI
    
    TrayIcon -->|Sends JSON| IPCServer
    IPCServer -->|Receives JSON| IPCClient
    IPCClient -->|Updates UI| MainWindow
    
    MainWindow -->|Sends JSON| IPCClient
    IPCClient -->|Receives JSON| IPCServer
    IPCServer -->|Calls set_config| SimpleModule
    IPCServer -->|Calls set_config| ExternalModule
```

---

## 6. Settings Persistence and Module Configuration

Configuration data flow and storage:

```mermaid
graph TB
    A["settings.json<br/>on disk"]
    B["SettingsAPI<br/>helpers"]
    C["Runner<br/>in-memory config"]
    D["Module<br/>object"]
    E["Settings UI<br/>ViewModel"]
    F["IPC Named Pipe<br/>JSON messages"]

    A -->|Read at startup| B
    B -->|Load to memory| C
    C -->|Query via get_config| D
    D -->|Return JSON| B
    D -->|Return to Settings UI| E
    
    E -->|User modifies| F
    F -->|Send new config| C
    C -->|Call set_config| D
    D -->|Apply changes| D
    C -->|Write to file| A
```

---

## 7. Hotkey Conflict Detection

How PowerToys detects and handles hotkey conflicts:

```mermaid
graph LR
    A["User sets<br/>hotkey in UI"]
    B["Settings UI queries<br/>Runner"]
    C["Runner checks<br/>module hotkeys"]
    D{"Conflict with<br/>other modules?"}
    E["Check Windows<br/>reserved hotkeys"]
    F{"System<br/>conflict?"}
    G["Check external app<br/>registry keys"]
    H{"External<br/>conflict?"}
    I["Return: Available"]
    J["Return: Conflict warning"]
    K["User sees warning<br/>in UI"]

    A --> B
    B --> C
    C --> D
    D -->|No| E
    D -->|Yes| J
    E --> F
    F -->|Yes| J
    F -->|No| G
    G --> H
    H -->|Yes| J
    H -->|No| I
    I --> K
    J --> K
```

---

## 8. Startup Sequence (Detailed)

Complete startup sequence with timing and dependencies:

```mermaid
graph TD
    A["PowerToys.exe<br/>entry point"]
    B["1. Initialize logger<br/>2. Create mutex<br/>3. Parse args"]
    C["Read settings.json<br/>from AppData"]
    D["Create tray icon<br/>window class"]
    E["Register with<br/>Windows taskbar"]
    F["Install low-level<br/>keyboard hook"]
    G["Enumerate known<br/>module DLL paths"]
    H["For each module:<br/>LoadLibrary & powertoy_create"]
    I["For each enabled<br/>module: call enable"]
    J["For each enabled<br/>module: call get_hotkeys"]
    K["Register all hotkeys<br/>with keyboard hook"]
    L["Enter Windows<br/>message loop"]
    M["Ready: waiting for<br/>input"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I --> J
    J --> K
    K --> L
    L --> M
```

---

## 9. Settings Change Propagation

How a setting change flows through the system:

```mermaid
graph LR
    A["User toggles<br/>setting in UI"]
    B["ViewModel<br/>updates"]
    C["IPC JSON<br/>message sent"]
    D["Runner receives<br/>message"]
    E["Parse JSON &<br/>identify module"]
    F["Write to<br/>settings.json"]
    G["Call module.<br/>set_config"]
    H["Module updates<br/>internal state"]
    I["If hotkey changed:<br/>update hook"]
    J["Next hotkey press<br/>uses new config"]

    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    E --> G
    G --> H
    H --> I
    I --> J
```

---

## 10. Process Architecture

Two main processes and their communication:

```mermaid
graph TB
    subgraph RunnerProc["PowerToys.exe (Runner Process)"]
        direction TB
        Main["main()/render_main()"]
        ModLoad["Module Loading"]
        MsgLoop["Message Loop"]
        HkHook["Keyboard Hook"]
        TrayUI["Tray Icon UI"]
        IPC1["IPC Named Pipe<br/>Server"]
    end

    subgraph SettingsProc["SettingsUI.exe (Settings Process)"]
        direction TB
        SettingsMain["Main Window"]
        ViewModels["ViewModels"]
        IPC2["IPC Named Pipe<br/>Client"]
        Panels["Module Settings<br/>Panels"]
    end

    Main --> ModLoad
    ModLoad --> MsgLoop
    MsgLoop --> HkHook
    MsgLoop --> TrayUI
    MsgLoop --> IPC1
    
    SettingsMain --> Panels
    Panels --> ViewModels
    ViewModels --> IPC2
    
    IPC1 <-->|JSON<br/>messages| IPC2
    
    TrayUI -.->|Launches| SettingsProc
```

---

## Summary

These diagrams illustrate:

1. **Runtime flow** – Startup through message loop
2. **Module loading** – DLL instantiation and initialization sequence
3. **Hotkey dispatch** – How keyboard input reaches modules
4. **Settings communication** – IPC between Runner and Settings UI
5. **Module architecture** – Different module types and their relationships
6. **Configuration storage** – Settings file, in-memory state, and module binding
7. **Conflict detection** – Hotkey validation before assignment
8. **Detailed startup** – Step-by-step initialization with dependencies
9. **Setting propagation** – How config changes flow through the system
10. **Process architecture** – Two main processes and their IPC channel

Each diagram emphasizes the key components and how they interact to achieve PowerToys' functionality.
