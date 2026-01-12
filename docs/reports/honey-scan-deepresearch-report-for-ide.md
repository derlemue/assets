# 🛸 Honey-Scan Architecture Map for Antigravity IDE

Diese Übersicht visualisiert die statische Struktur und die dynamischen Prozesse des Projekts, um die Navigation und das Verständnis innerhalb der IDE zu erleichtern.

## 1. Modul-Abhängigkeits-Graph (Static Structure)
Dieses Klassendiagramm zeigt die Beziehungen zwischen den Python-Modulen. Es hilft der IDE, Import-Pfade und Zuständigkeiten zu verstehen.

```mermaid
classDiagram
    %% Definition der Klassen/Module
    class Main {
        +main()
        +parse_args()
    }
    
    class Controller {
        -queue: TargetQueue
        -threads: List
        +init_scan(config)
        +dispatch_workers()
        +collect_results()
    }
    
    class Scanner {
        +target_ip: str
        +port: int
        +timeout: int
        +connect()
        +send_payload()
        +get_banner()
    }
    
    class SignatureDB {
        +patterns: dict
        +load_from_json()
        +check_match(data)
    }
    
    class Logger {
        +log_file: str
        +write_json(data)
        +print_console(msg)
    }

    %% Beziehungen
    Main ..> Controller : instantiates
    Main ..> Logger : uses
    Controller --> Scanner : spawns (1..n)
    Controller ..> Logger : reports to
    Scanner ..> SignatureDB : queries
```

---

## 2. Prozess-Ablauf: Runtime Logic
Dieser Flowchart visualisiert den Algorithmus von der Ingest-Phase bis zur Egress-Phase. Er zeigt, wo Entscheidungen getroffen werden (Branches).

```mermaid
graph TD
    %% Nodes
    Start([🚀 Entry Point: main.py])
    
    subgraph "Phase 1: Configuration"
        Init[Parse CLI Args]
        LoadDB[Load signatures.json]
        TargetGen[Generate Target List]
    end
    
    subgraph "Phase 2: The Loop (controller.py)"
        CheckQueue{Queue Empty?}
        SpawnWorker[Spawn Thread / Async Task]
        JoinThreads[Wait for Completion]
    end
    
    subgraph "Phase 3: Deep Scan (scanner.py)"
        Connect[Socket Connect]
        Send[Send Honeypot Payload]
        Recv[Receive Banner/Response]
        
        Analyze{Match Signature?}
        Analyze -- Yes --> ScoreHigh[Set Honeyscore: 1.0]
        Analyze -- No --> Heuristic{Heuristic Check?}
        
        Heuristic -- Suspicious --> ScoreMed[Set Honeyscore: 0.6]
        Heuristic -- Clean --> ScoreLow[Set Honeyscore: 0.0]
    end
    
    subgraph "Phase 4: Output (logger.py)"
        Format[Format Result Dict]
        Save[Write to JSON/Console]
    end

    %% Edges
    Start --> Init
    Init --> LoadDB
    LoadDB --> TargetGen
    TargetGen --> CheckQueue
    
    CheckQueue -- No --> SpawnWorker
    SpawnWorker --> Connect
    
    Connect -->|Error| ScoreLow
    Connect -->|Success| Send
    Send --> Recv
    Recv --> Analyze
    
    ScoreHigh --> Format
    ScoreMed --> Format
    ScoreLow --> Format
    
    Format --> CheckQueue
    CheckQueue -- Yes --> JoinThreads
    JoinThreads --> Save
    Save --> End([🏁 Exit])
```

---

## 3. Data Flow & Transformation (ETL)
Für die IDE ist es wichtig zu wissen, wie sich die Datenstrukturen verändern. Dieses Diagramm zeigt die Metamorphose vom Raw-Input zum Report.

```mermaid
flowchart LR
    %% Data States
    InputState(Raw Input String)
    TargetObj(Target Object)
    NetworkBytes(Raw Bytes)
    CleanStr(Normalized String)
    ResultDict(Result Dictionary)
    FinalJSON(JSON Report)

    %% Processes
    P1[[ArgParse]]
    P2[[Socket I/O]]
    P3[[Decoding/Strip]]
    P4[[Regex Logic]]
    P5[[Serialization]]

    %% Flow
    InputState --> P1 --> TargetObj
    TargetObj --> P2 --> NetworkBytes
    NetworkBytes --> P3 --> CleanStr
    CleanStr --> P4 --> ResultDict
    ResultDict --> P5 --> FinalJSON

    %% File Mapping Context
    subgraph Context [File Mapping]
        P1 -.-> main.py
        P2 -.-> scanner.py
        P3 -.-> scanner.py
        P4 -.-> signatures.py
        P5 -.-> logger.py
    end
```

---

## 4. Zustandsautomat (State Machine) für ein Scan-Target
Der Lebenszyklus eines einzelnen Ziels (Host) innerhalb der Applikation.

```mermaid
stateDiagram-v2
    [*] --> QUEUED : main.py
    
    QUEUED --> CONNECTING : scanner.py
    
    state CONNECTING {
        [*] --> HANDSHAKE
        HANDSHAKE --> PAYLOAD_SENT
        PAYLOAD_SENT --> WAITING_RESPONSE
    }
    
    CONNECTING --> FAILED : Timeout/Refused
    WAITING_RESPONSE --> ANALYZING : Data Received
    
    state ANALYZING {
        [*] --> PATTERN_MATCHING
        PATTERN_MATCHING --> HEURISTICS
    }
    
    ANALYZING --> HONEYPOT_DETECTED : Match Found
    ANALYZING --> CLEAN_HOST : No Match
    
    HONEYPOT_DETECTED --> REPORTING
    CLEAN_HOST --> REPORTING
    FAILED --> REPORTING
    
    REPORTING --> [*] : Written to Log
```
