# Honey Scan: Technischer Analysebericht & Architektur-Roadmap

**Status:** Draft / Deep Research  
**Datum:** 13.01.2026  
**Autor:** Senior Lead Architect  
**Repository:** `derlemue/honey-scan` (Fork von `hacklcx/HFish`)

---

## 1. Executive Summary

Dieser Bericht analysiert den aktuellen Zustand des honey-scan Repositories, bewertet die technische Machbarkeit eines Wechsels zu PHP und definiert die zukünftige hybride Architektur. Die Analyse bestätigt, dass eine vollständige Portierung des Backends auf PHP abgelehnt (**VETO**) wird, da dies die Performance bei High-Concurrency-Angriffen gefährdet. Stattdessen wird eine hybride Strategie (Go Core + Web UI) verfolgt.

---

## 2. Ist-Zustand: Codebase & "Localization Debt"

Das Repository ist ein Fork von HFish, einer chinesischen Honeypot-Plattform. Die größte technische Schuld (Technical Debt) liegt in der tiefen Verwurzelung der chinesischen Sprache in Datenbank-Schemata, Kommentaren und hartcodierter Logik.

### 2.1 Visuelle Code-Analyse (Diff)

Das folgende Diagramm zeigt die Diskrepanz zwischen dem Original und dem aktuellen Fork sowie die Verteilung der Altlasten.

```mermaid
graph TD
    subgraph "Legacy HFish (Original)"
        A["Original Core (Go)"] -->|Enthält| B("CN Hardcoded Strings")
        A -->|Nutzt| C("CN DB Schema 'yonghu', 'mima'")
        A -->|Logik| D("CN Cloud Connectors")
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

```

### 2.2 Verteilung der Komponenten

Diese Übersicht zeigt, wo sich die sprachlichen und logischen Barrieren befinden.

```mermaid
pie title "Verteilung der 'Localization Debt' im Repository"
    "Englisches UI (Refactored)" : 30
    "Englisches Backend (Refactored)" : 10
    "CN Datenbank-Schema (Legacy)" : 25
    "CN Kommentare & Docs (Legacy)" : 15
    "CN Hardcoded Logic (Legacy)" : 20

```

---

## 3. Das Performance-Dilemma: Go vs. PHP

Ein Kernpunkt der Anforderung war die Evaluierung eines Wechsels zu PHP für das Backend.

### 3.1 Architektur-Entscheidung: VETO für PHP im Core

Die folgende Sequenzanalyse verdeutlicht, warum PHP für einen High-Interaction Honeypot ungeeignet ist (Blocking I/O vs. Non-Blocking I/O).

**Szenario:** 10.000 gleichzeitige SYN-Requests (Angriffswelle)

```mermaid
sequenceDiagram
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

    rect rgb(255, 200, 20)
