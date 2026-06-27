# Power Display diagrams

## 1. Runtime flow

```mermaid
graph TD
    A[PowerToys runner] --> B[PowerDisplayModuleInterface DLL]
    B -->|enable| C[PowerDisplayProcessManager]
    C -->|launches with runner PID + pipe name| D[PowerToys.PowerDisplay.exe]
    D --> E[Program.Main]
    E --> F[App.OnLaunched]
    F --> G[MainWindow]
    G --> H[MainViewModel]
    H --> I[MonitorManager]
    I --> J[DisplayConfigInventory]
    J --> K[WMI controller]
    J --> L[DDC/CI controller]
    K --> M[MonitorViewModels]
    L --> M
    M --> N[Brightness / display controls]
```

## 2. Module and file interaction

```mermaid
graph LR
    subgraph Runner side
        R[PowerToys runner]
        DLL[dllmain.cpp]
        PM[PowerDisplayProcessManager.cpp]
    end

    subgraph PowerDisplay app
        P[Program.cs]
        APP[App.xaml.cs]
        MW[MainWindow]
        VM[MainViewModel*.cs]
        MVM[MonitorViewModel.cs]
        MM[MonitorManager.cs]
    end

    subgraph Display integration
        DCI[DisplayConfigInventory.cs]
        WMI[WmiController.cs]
        DDC[DdcCiController.cs]
        ID[MonitorIdentity.cs]
        MODEL[Monitor.cs]
    end

    subgraph Settings
        PDS[PowerDisplaySettings.cs]
        PDVM[PowerDisplayViewModel.cs]
        PROF[ProfileService / MonitorStateManager]
    end

    R --> DLL --> PM --> P --> APP --> MW --> VM --> MM
    VM --> MVM --> MM
    MM --> DCI
    MM --> WMI
    MM --> DDC
    DCI --> ID
    WMI --> ID
    WMI --> MODEL
    DDC --> MODEL
    PDVM -->|settings + events| APP
    VM --> PDS
    VM --> PROF
```

## 3. Sequence: changing monitor brightness

```mermaid
sequenceDiagram
    participant User
    participant UI as Monitor card / slider
    participant MVM as MonitorViewModel
    participant MM as MonitorManager
    participant CTRL as Selected controller
    participant OS as WMI or DDC/CI API
    participant State as Settings / state persistence

    User->>UI: Change brightness
    UI->>MVM: Set Brightness property
    MVM->>MVM: Debounce commit
    MVM->>MM: SetBrightnessAsync(monitorId, value)
    MM->>MM: Lookup Monitor by Id
    MM->>MM: Select controller from CommunicationMethod
    MM->>CTRL: SetBrightnessAsync(monitor, value)
    CTRL->>OS: WmiSetBrightness or VCP 0x10 write
    OS-->>CTRL: Success or failure
    CTRL-->>MM: MonitorOperationResult
    alt success
        MM->>MM: Update Monitor.CurrentBrightness
        MVM->>State: Save monitor setting
    else failure
        MVM->>MVM: Log warning/error
    end
```

## 4. Sequence: monitor discovery

```mermaid
sequenceDiagram
    participant VM as MainViewModel
    participant MM as MonitorManager
    participant DC as DisplayConfigInventory
    participant WMI as WmiController
    participant DDC as DdcCiController
    participant UI as MonitorViewModels

    VM->>MM: DiscoverMonitorsAsync()
    MM->>DC: GetAllMonitorDisplayInfo()
    DC-->>MM: Active display inventory
    MM->>MM: Filter built-in blacklist
    alt inventory empty
        MM-->>VM: Empty monitor list
    else active displays found
        MM->>WMI: DiscoverMonitorsAsync(all displays)
        WMI-->>MM: WMI-managed monitors
        MM->>MM: Partition displays not claimed by WMI
        MM->>DDC: DiscoverMonitorsAsync(DDC targets)
        DDC-->>MM: DDC/CI-managed monitors
        MM-->>VM: Sorted combined monitor list
        VM->>UI: Rebuild visible monitor cards
    end
```

## 5. Controller routing overview

```mermaid
graph TD
    A[Active display from DisplayConfig] --> B{WMI exposes matching brightness instance?}
    B -->|Yes| C[Create WMI monitor]
    B -->|No| D[Route to DDC/CI]
    D --> E{Logical hMonitor and physical monitor handles available?}
    E -->|No| F[No DDC monitor produced]
    E -->|Yes| G{Capabilities or max-compat probe finds VCP features?}
    G -->|No| H[No DDC monitor or no brightness support]
    G -->|Yes| I[Create DDC/CI monitor]
    C --> J[CommunicationMethod = WMI]
    I --> K[CommunicationMethod = DDC/CI]
    J --> L[Brightness writes use WMI]
    K --> M[Brightness writes use VCP 0x10]
```
