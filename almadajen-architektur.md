# Almadajen – Architektur der App

Wie die App für Geflügelfarmen intern arbeitet: von der Eingabe des Arbeiters im Stall in Syrien bis zur Push-Nachricht auf dem Handy des Besitzers in Deutschland. Stand 24. November 2026, Version 0.25.13.

`Flutter` · `Firestore offline-first` · `Cloud Functions v2` · `FCM` · `europe-west3` · `Claude AI`

[← zurück zum Profil](README.md#almadajen--app-für-geflügelfarmen)

## 1. Gesamtarchitektur

Eine Flutter-App läuft auf Android (Google Play oder APK von der Website) und als Web-App auf iPhone und PC. Alle Geräte schreiben in Firestore; der Server rechnet und verschickt Warnungen.

```mermaid
flowchart LR
  subgraph USERS["Nutzer"]
    W["Arbeiter<br/>nur zugewiesene Durchgänge"]
    O["Besitzer"]
    S["Betreuer / Tierarzt<br/>nur lesen"]
    OP["Betreiber<br/>Farmverwaltung"]
  end
  subgraph APP["Flutter-App"]
    AND["Android<br/>Google Play + APK"]
    WEB["Web-App<br/>iPhone + PC"]
    CACHE[("Speicher im Handy<br/>funktioniert offline")]
  end
  subgraph FB["Firebase · europe-west3"]
    AUTH["Authentication<br/>Rollen als Claims"]
    FS[("Firestore<br/>+ Security Rules")]
    FN["Cloud Functions v2"]
    FCM["FCM<br/>Push-Nachrichten"]
    HOST["Hosting<br/>Website und APK-Download"]
  end
  subgraph EXT["Externe Dienste"]
    AI["Claude AI<br/>Analyse und Bilderkennung"]
    MET["Open-Meteo<br/>Wettervorhersage"]
  end
  W --> AND
  O --> AND
  O --> WEB
  S --> WEB
  OP --> AND
  AND --> CACHE
  WEB --> CACHE
  CACHE <-->|"Synchronisierung"| FS
  AND --> AUTH
  WEB --> AUTH
  FS --> FN
  FN --> FS
  FN --> FCM
  FCM -->|"Warnung"| AND
  FCM -->|"Warnung"| WEB
  FN <--> AI
  FN <--> MET
  HOST -.->|"Update"| AND
  HOST -->|"lädt"| WEB
```

## 2. Datenfluss: von der Eingabe zur Warnung

Der Arbeiter ändert nie alte Daten; jede Eingabe ist ein neues Ereignis. Der Server fasst die Ereignisse eines Tages zusammen, vergleicht sie mit dem Rassenleitfaden für das Alter der Tiere und schickt bei Bedarf eine Warnung.

```mermaid
sequenceDiagram
  autonumber
  participant W as Arbeiter
  participant P as Handy (offline)
  participant F as Firestore
  participant S as onOpsEventCreated
  participant A as Warnregeln
  participant N as onAlertWritten + FCM
  participant O as Besitzer
  W->>P: Verluste, Futter, Wasser, Temperatur
  P->>P: sofort lokal gespeichert
  P-->>F: gesendet, sobald Netz da ist (ops_events)
  F->>S: neues Ereignis
  S->>F: daily_summaries für diesen Tag
  S->>A: Vergleich mit Leitfaden Ross / Cobb / Hy-Line
  A->>F: alerts (hohe Verluste, zu wenig Wasser ...)
  F->>N: neue Warnung
  N->>O: Push-Nachricht aufs Handy
```

## 3. Geplante Server-Jobs und KI

Einige Funktionen laufen zu festen Zeiten, andere startet der Nutzer mit einem Knopf in der App.

```mermaid
flowchart TB
  subgraph SCHED["Zeitgesteuert"]
    H["heatForecastCheck<br/>Hitze- und Kältewarnung für morgen, pro Standort"]
    SB["silentBarnCheck<br/>Stall ohne Eingabe"]
    SUB["subscriptionCheck<br/>Abo läuft ab"]
    WK["weeklyAiAnalysis<br/>Wochenanalyse"]
    DM["demoDailyFill<br/>Daten der Demo-Farm"]
    BK["Tägliches Backup<br/>nur prod, 7 Tage"]
  end
  subgraph BTN["Per Knopf in der App"]
    DG["aiDiagnose<br/>Frage zu kranken Tieren"]
    LB["aiReadLabel<br/>Foto der Medikamentenpackung"]
    TH["aiReadThermometer<br/>Foto des Thermometers"]
    AC["createWorker / revokeWorker<br/>resetMemberPassword"]
    PA["platformAdmin<br/>Farmverwaltung"]
  end
  H --> ALR[("alerts")]
  SB --> ALR
  ALR -->|"onAlertWritten"| PUSH["FCM<br/>Push-Nachricht"]
  WK --> CL["Claude AI"]
  WK --> AIA[("ai_analyses<br/>Karte im Bericht")]
  WK -->|"an Besitzer + Betreuer"| PUSH
  SUB -->|"an Besitzer + Betreiber"| PUSH
  DG --> CL
  LB --> CL
  TH --> CL
```

## 4. Vom Code zum Nutzer

Jede Änderung wird zuerst im Testprojekt ausprobiert und dann ins echte Projekt übernommen. Rules und Functions gehen vor der App live.

```mermaid
flowchart LR
  CODE["Code<br/>app + functions"] --> TEST["Tests<br/>flutter test · npm test · Rules"]
  TEST --> REV["Unabhängiges Review"]
  REV --> GH[("GitHub<br/>privates Backup")]
  REV --> STG["Deploy Rules + Functions<br/>poultry-farm-staging"]
  STG --> PRD["Deploy Rules + Functions<br/>poultry-farm-prod"]
  PRD --> REL["release.ps1 -Play"]
  REL --> SITES["Beide Websites<br/>APK + Web-App"]
  REL --> AAB["AAB für Google Play"]
  AAB --> PLAY["Prüfung durch Google<br/>dann Veröffentlichung"]
```

## Begriffe

- **ops_events**: Jede Eingabe des Arbeiters wird als einzelnes Ereignis gespeichert. Nichts wird gelöscht; auch eine Korrektur ist ein neues Ereignis.
- **daily_summaries**: Zusammenfassung pro Tag und Durchgang: Verluste, Futter nach Sorte, Wasser, Temperatur. Der Server berechnet sie.
- **Tag 1**: Ankunftstag der Küken (Alter 0 Tage).
- **strains.ts**: Die offiziellen Rassentabellen. Daraus entsteht `assets/strains.json` für die App; Tests prüfen, dass beide gleich sind.
- **Rollen**: Der Besitzer sieht alles, der Betreuer liest nur, der Arbeiter sieht nur seine Durchgänge, der Betreiber verwaltet Farmen und Abos.
