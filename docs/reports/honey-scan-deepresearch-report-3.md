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
