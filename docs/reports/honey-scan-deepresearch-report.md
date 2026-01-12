📊 Prozess-Visualisierung: Honey-ScanDiese Diagramme visualisieren die Logik und den Datenfluss basierend auf der Analyse des Codes.1. Der Haupt-Workflow (Orchestrierung)Dieses Diagramm zeigt, wie der Controller die Aufgaben verteilt. Es ist in logische Phasen (Subgraphen) unterteilt.graph TD
    %% Styling
    classDef init fill:#2d3436,stroke:#fff,stroke-width:2px,color:#fff;
    classDef process fill:#0984e3,stroke:#fff,color:#fff;
    classDef decision fill:#e17055,stroke:#fff,color:#fff;
    classDef storage fill:#00b894,stroke:#fff,color:#fff;

    subgraph "Phase 1: Initialisierung"
        Start((🚀 Start)):::init --> Args[CLI Argumente parsen]:::process
        Args --> Config[Konfiguration laden]:::process
        Config --> Targets{Ziele definiert?}:::decision
        Targets -- Nein --> Error[Abbruch]:::decision
        Targets -- Ja --> Queue[Target Queue füllen]:::storage
    end

    subgraph "Phase 2: Execution (Multi-Threading/Async)"
        Queue --> Dispatcher{Worker verfügbar?}:::decision
        Dispatcher -- Ja --> Worker[Spawn Scanner Worker]:::process
        
        subgraph "Subprozess: Scan Unit"
            Worker --> Connect[Socket Connect]:::process
            Connect --> Payload[Sende 'Fake' Header]:::process
            Payload --> Receive[Empfange Antwort]:::process
            Receive --> Analyze[Signatur Vergleich]:::process
        end
        
        Analyze --> Result[Ergebnis Objekt]:::process
    end

    subgraph "Phase 3: Aggregation & Output"
        Result --> Aggregator[Ergebnisse sammeln]:::storage
        Dispatcher -- Queue leer --> Export{Format?}:::decision
        Export -- JSON --> WriteJSON[Schreibe .json]:::storage
        Export -- CLI --> PrintOut[Konsole Ausgabe]:::process
        WriteJSON --> End((🏁 Ende)):::init
        PrintOut --> End
    end
2. Detail-Logik: Der "Fingerprinting" SubprozessHier zoomen wir tief in die Funktion hinein, die entscheidet, ob eine IP ein Honeypot ist. Dies visualisiert die If/Else-Logik im Scanner-Modul.flowchart TD
    %% Nodes
    Input([Eingabe: IP & Port]) --> SocketInit[Initialisiere Socket]
    SocketInit --> TryConnect{Verbindung möglich?}
    
    %% Connection Logic
    TryConnect -- Timeout/Refused --> DeadHost[❌ Host Offline / Port zu]
    TryConnect -- Connected --> SendBytes[Sende Protokoll-Payload]
    
    %% Interaction Logic
    SendBytes --> WaitResp{Antwort erhalten?}
    WaitResp -- Timeout --> GenericCheck[Prüfe auf TCP-Verhalten]
    WaitResp -- Data Received --> Decode[Decode Bytes to String]
    
    %% Analysis Logic
    Decode --> MatchDB{Match in Signatures.py?}
    
    MatchDB -- Ja (Kippo/Cowrie) --> SetFlag[⚠️ Setze Flag: HONEYPOT]
    MatchDB -- Nein --> CheckHex[Prüfe Hex-Muster]
    
    CheckHex -- Match --> SetFlag
    CheckHex -- Kein Match --> CleanHost[✅ Status: Clean System]
    
    %% Return
    SetFlag --> Return([Return Result Object])
    CleanHost --> Return
    DeadHost --> Return
    GenericCheck --> Return

    %% Styling
    style SetFlag fill:#d63031,stroke:#fff,color:#fff
    style CleanHost fill:#00b894,stroke:#fff,color:#fff
    style MatchDB fill:#fdcb6e,stroke:#333,color:#000
3. Datenstrom & Objekt-Zustände (State Diagram)Dieses Diagramm zeigt, welche Zustände ein "Target" (eine Ziel-IP) während der Laufzeit des Programms durchläuft.stateDiagram-v2
    [*] --> Pending: Zur Warteschlange hinzugefügt
    
    Pending --> Scanning: Worker übernimmt Job
    
    state Scanning {
        [*] --> Connecting
        Connecting --> Handshaking: SYN-ACK erhalten
        Handshaking --> AwaitingResponse: Payload gesendet
        AwaitingResponse --> Processing: Daten empfangen
    }
    
    Processing --> Classified: Signatur gefunden (Honeypot)
    Processing --> Cleared: Keine Signatur (Legit)
    Scanning --> Error: Timeout/Connection Refused
    
    Classified --> Archiviert: In JSON geschrieben
    Cleared --> Archiviert: In JSON geschrieben
    Error --> Verworfen: Aus Logik entfernt
    
    Archiviert --> [*]
    Verworfen --> [*]
4. API Sequenz (Kommunikation)Wer spricht mit wem? Dieses Diagramm zeigt die Abhängigkeiten zwischen den Python-Modulen/Klassen.sequenceDiagram
    participant User
    participant Main as main.py
    participant Controller as Controller Class
    participant Scanner as Scanner Module
    participant Network as Target IP (Remote)
    participant DB as Signatures DB

    User->>Main: Startet Script (-t 192.168.1.5)
    Main->>Controller: init(target)
    Controller->>Controller: create_thread_pool()
    
    loop Für jeden Port
        Controller->>Scanner: scan_host(ip, port)
        activate Scanner
        
        Scanner->>Network: Socket Open (SYN)
        Network-->>Scanner: Socket Accept (ACK)
        
        Scanner->>Network: Send "SSH-2.0-Fake"
        Network-->>Scanner: Response Bytes
        
        Scanner->>DB: get_patterns("ssh")
        DB-->>Scanner: return regex_list
        
        rect rgb(30, 30, 40)
            note right of Scanner: Matching Prozess
            Scanner->>Scanner: regex.match(Response)
        end
        
        Scanner-->>Controller: return Result(is_honeypot=True)
        deactivate Scanner
    end

    Controller->>User: Print Ergebnis (Console/File)
