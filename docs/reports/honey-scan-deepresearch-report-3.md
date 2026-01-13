Honey Scan: Technischer Analysebericht & Architektur-RoadmapStatus: Draft / Deep ResearchDatum: 13.01.2026Autor: Senior Lead ArchitectRepository: derlemue/honey-scan (Fork von hacklcx/HFish)1. Executive SummaryDieser Bericht analysiert den aktuellen Zustand des honey-scan Repositories, bewertet die technische Machbarkeit eines Wechsels zu PHP und definiert die zukünftige hybride Architektur. Die Analyse bestätigt, dass eine vollständige Portierung des Backends auf PHP abgelehnt (VETO) wird, da dies die Performance bei High-Concurrency-Angriffen gefährdet. Stattdessen wird eine hybride Strategie (Go Core + Web UI) verfolgt.2. Ist-Zustand: Codebase & "Localization Debt"Das Repository ist ein Fork von HFish, einer chinesischen Honeypot-Plattform. Die größte technische Schuld (Technical Debt) liegt in der tiefen Verwurzelung der chinesischen Sprache in Datenbank-Schemata, Kommentaren und hartcodierter Logik.2.1 Visuelle Code-Analyse (Diff)Das folgende Diagramm zeigt die Diskrepanz zwischen dem Original und dem aktuellen Fork sowie die Verteilung der Altlasten.graph TD
    subgraph "Legacy HFish (Original)"
        A[Original Core (Go)] -->|Enthält| B(CN Hardcoded Strings)
        A -->|Nutzt| C(CN DB Schema 'yonghu', 'mima')
        A -->|Logik| D(CN Cloud Connectors)
    end

    subgraph "Honey Scan (Current Fork)"
        E[Fork Core]
        F[Web UI]
    end

    A -.->|Forked| E
    
    E -->|Refactoring Status: 10%| G{Backend Logic}
    F -->|Translation Status: 30%| H{Frontend UI}

    G -- "Critical Debt" --> B
    G -- "Critical Debt" --> C
    
    style B fill:#ffcccc,stroke:#333
    style C fill:#ffcccc,stroke:#333
    style E fill:#e6f3ff,stroke:#333
    style G fill:#fff2cc,stroke:#333
2.2 Verteilung der KomponentenDiese Übersicht zeigt, wo sich die sprachlichen und logischen Barrieren befinden.pie title "Verteilung der 'Localization Debt' im Repository"
    "Englisches UI (Refactored)" : 30
    "Englisches Backend (Refactored)" : 10
    "CN Datenbank-Schema (Legacy)" : 25
    "CN Kommentare & Docs (Legacy)" : 15
    "CN Hardcoded Logic (Legacy)" : 20
3. Das Performance-Dilemma: Go vs. PHPEin Kernpunkt der Anforderung war die Evaluierung eines Wechsels zu PHP für das Backend.3.1 Architektur-Entscheidung: VETO für PHP im CoreDie folgende Sequenzanalyse verdeutlicht, warum PHP für einen High-Interaction Honeypot ungeeignet ist (Blocking I/O vs. Non-Blocking I/O).Szenario: 10.000 gleichzeitige SYN-Requests (Angriffswelle)sequenceDiagram
    participant Attacker as Angreifer (Botnet)
    participant Go as Go Routine (Honeypot)
    participant PHP as PHP Worker (Alternative)
    participant Sys as System Resources

    Note over Attacker, Go: Vergleich der Lastbewältigung

    rect rgb(200, 255, 200)
    Note right of Go: Go Architektur (Non-Blocking)
    Attacker->>Go: Sendet 10k Pakete
    Go->>Go: Spawn 10k Goroutines (Micro-Threads)
    Go->>Sys: Verbraucht ~50MB RAM
    Go-->>Attacker: Akzeptiert Verbindungen (Keep-Alive)
    end

    rect rgb(255, 200, 200)
    Note right of PHP: PHP Architektur (Request-Response)
    Attacker->>PHP: Sendet 10k Pakete
    loop Für jeden Request
        PHP->>Sys: Startet Prozess / Thread
        PHP->>Sys: Lädt Framework & Config
        PHP--xSys: RAM Limit erreicht / Prozess-Limit!
    end
    PHP-->>Attacker: Connection Refused / Timeout
    end
Ergebnis: Go kann durch Goroutines tausende Verbindungen halten. PHP muss für jeden Request den gesamten Kontext laden (Bootstrapping), was bei Flood-Angriffen zum Kollaps führt.4. Soll-Architektur: Hybrider AnsatzUm den Wunsch nach einfacherer Anpassbarkeit (PHP/Web) mit der notwendigen Performance (Go) zu vereinen, wird eine strikte Trennung eingeführt.4.1 System-Architektur Diagramm (C4 Level 2)flowchart TB
    subgraph "External Threat Landscape"
        Attacker[Angreifer / Scanner]
    end

    subgraph "Honey Scan Server Node"
        direction TB
        
        subgraph "Performance Layer (Go)"
            Listener[Sensor Core (Go)]
            Parser[Protocol Parser]
            API_Go[Internal REST API]
        end

        subgraph "Data Layer (Docker)"
            DB[(MySQL / SQLite)]
            Redis[(Redis Queue)]
        end

        subgraph "Management Layer (PHP/Python/Web)"
            WebUI[Admin Dashboard]
            Config[Config Generator]
            Reporter[Report Engine]
        end
    end

    Attacker -- "TCP/UDP Connect" --> Listener
    Listener --> Parser
    Parser -- "Normalized JSON" --> Redis
    
    Redis -- "Async Write" --> DB
    
    WebUI -- "Read Data" --> DB
    WebUI -- "Push Config (JSON)" --> API_Go
    API_Go -- "Reload Services" --> Listener

    style Listener fill:#bbf,stroke:#333,stroke-width:2px
    style WebUI fill:#bfb,stroke:#333,stroke-width:2px
    style Attacker fill:#f99,stroke:#333
5. Detaillierte ProzessabläufeHier zoomen wir in die spezifischen Abläufe hinein, die während des Refactorings implementiert werden müssen.5.1 Prozess: De-Localization & Schema Migration (Phase 1 & 3)Wie wandeln wir die chinesischen Daten in englische Strukturen um?flowchart LR
    A[Start: Legacy DB] --> B{Schema Prüfung}
    
    B -- "Tabelle: yonghu" --> C[Rename: users]
    B -- "Spalte: mima" --> D[Rename: password]
    B -- "Tabelle: gongji" --> E[Rename: attacks]
    
    C --> F[Data Migration Script]
    D --> F
    E --> F
    
    F --> G[(Neue DB Struktur)]
    G --> H[Update Go Structs]
    G --> I[Update Web UI Models]
    
    style A fill:#ffcccc
    style G fill:#ccffcc
5.2 Prozess: Datenfluss vom Sensor zum Dashboard (Detail)Dieser Flow zeigt, wie Daten künftig fließen, ohne dass das Dashboard (PHP) den Sensor (Go) blockiert.sequenceDiagram
    participant S as Sensor (Go)
    participant Q as Message Queue (Redis)
    participant W as Worker (Python/Go)
    participant D as Datenbank
    participant U as Admin UI (PHP)

    S->>S: Erkennt Angriff (Port 22)
    S->>S: Extrahiert Metadaten (IP, Payload)
    
    Note over S, Q: Fire & Forget (Non-Blocking)
    S->>Q: Push JSON Event
    
    loop Async Processing
        W->>Q: Pop Event
        W->>W: GeoIP Anreicherung
        W->>W: Attack Signature Matching
        W->>D: Insert in 'attacks' Tabelle
    end

    U->>D: SELECT * FROM attacks WHERE time > 5m
    D-->>U: Result Set
    U-->>User: Zeige Echtzeit-Dashboard
6. Implementation RoadmapDer Zeitplan für die Umsetzung der oben gezeigten Architektur.gantt
    title Honey Scan Modernisierungs-Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 1: Cleanup
    Audit Codebase (CN Strings)       :active, a1, 2026-01-15, 7d
    Refactor DB Schema (Englisch)     :a2, after a1, 10d
    Git History Cleanup               :a3, after a2, 3d

    section Phase 2: Architecture
    Decouple UI from Binary           :crit, b1, 2026-02-05, 14d
    Implement Internal API (Go)       :b2, after b1, 10d
    Setup Docker Compose Stack        :b3, after b2, 5d

    section Phase 3: Migration
    Data Migration Scripts            :c1, 2026-03-05, 7d
    Performance Tuning                :c2, after c1, 7d
    Release v2.0 (English)            :milestone, 2026-03-20, 0d
