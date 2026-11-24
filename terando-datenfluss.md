# Terando – Datenfluss und Sicherheit

Was der Terando-Server mit dem Börsenkonto macht: Anmeldung, API-Schlüssel, Handel, stündliche Schlüsselprüfung und Notfall. Stand 24. November 2026.

`Flutter` · `Python / FastAPI` · `HTTPS` · `verschlüsselte API-Schlüssel` · `IP-Beschränkung` · `Server in Nürnberg (EU)`

[← zurück zum Profil](README.md#terando--trading-bots-mit-handy-app)

Kurz: Der Server holt Daten nur über den API-Schlüssel, den der Nutzer selbst auf seiner Börse erstellt. Damit kann er handeln, aber **kein Geld abheben oder überweisen**. Einen Schlüssel mit diesem Recht lehnt er ab, beim Verbinden und danach jede Stunde.

## 1. Anmeldung und Schlüssel verbinden

Der Nutzer meldet sich mit Passwort und 2FA-Code an. Den API-Schlüssel erstellt er selbst auf der Börse. Der Server fragt die Börse nach den echten Rechten des Schlüssels und speichert ihn nur, wenn er handeln, aber nicht auszahlen kann.

```mermaid
sequenceDiagram
    autonumber
    actor N as Nutzer
    participant A as Handy-App
    participant S as Terando-Server
    participant B as Börse (z. B. Binance)

    rect rgba(15,122,90,0.08)
    Note over N,S: Anmeldung
    N->>A: E-Mail + Passwort
    A->>S: Anmeldung (HTTPS)
    S->>S: Passwort mit scrypt-Hash prüfen
    S-->>A: 2FA-Code nötig
    N->>A: 2FA-Code
    A->>S: Code (HTTPS)
    alt Passwort oder Code falsch
        S-->>A: Abgelehnt, nach mehreren Fehlversuchen gesperrt
    else Richtig
        S-->>A: Sitzungs-Token
    end
    end

    rect rgba(15,122,90,0.08)
    Note over N,B: API-Schlüssel verbinden
    N->>B: Schlüssel selbst erstellen: Lesen + Handel, IP = Server
    N->>A: Schlüssel eintragen
    A->>S: Schlüssel (HTTPS)
    S->>B: Welche Rechte hat dieser Schlüssel? (HTTPS)
    B-->>S: Echte Rechte des Schlüssels
    alt Auszahlung erlaubt oder ohne IP-Sperre
        S-->>A: Abgelehnt, Schlüssel wird verworfen
    else Nur Lesen + Handel, IP-gesperrt
        S->>S: Verschlüsselt speichern
        S-->>A: Verbunden (nur letzte 4 Zeichen sichtbar)
    end
    end
```

## 2. Handel und stündliche Schlüsselprüfung

Vor jeder Order prüft der Server Coin, Betrag, Liquidität und ob schon ein anderer Bot einen offenen Trade in diesem Coin hat. Jede Stunde fragt er die Rechte jedes echten Schlüssels erneut ab. Er reagiert nur auf eine eindeutige Antwort der Börse: Ein Timeout ändert nichts.

```mermaid
sequenceDiagram
    autonumber 17
    actor N as Nutzer
    participant A as Handy-App
    participant S as Terando-Server
    participant B as Börse (z. B. Binance)

    rect rgba(15,122,90,0.08)
    Note over S,B: Handel (laufend)
    S->>B: Kurse + Kontostand abfragen (HTTPS)
    B-->>S: Daten
    S->>S: Strategie erzeugt Signal (z. B. Trend, Safety-Stufe)
    S->>S: Prüfen: gewählter Coin? Betrag im Limit? Liquidität? Coin frei?
    S->>B: Kauf- oder Verkaufsorder (HTTPS)
    B-->>S: Ausgeführt (Geld bleibt auf dem Börsenkonto)
    S-->>A: Push-Meldung
    end

    rect rgba(15,122,90,0.08)
    Note over S,B: Schlüsselprüfung (jede Stunde)
    S->>B: Rechte des Schlüssels erneut abfragen (HTTPS)
    B-->>S: Echte Rechte
    alt Auszahlung jetzt erlaubt
        S->>S: Keine neuen Trades, offene laufen weiter
        S-->>A: Warnung: Auszahlung ausschalten
        Note over S,B: Nach 24 h noch erlaubt: Börse trennen, offene Orders stornieren
    else IP-Sperre oder Handelsrecht fehlt
        S->>S: Keine neuen Trades, bis behoben
        S-->>A: Warnung
    else Keine klare Antwort
        S->>S: Nichts ändern, nächste Stunde erneut
    end
    end
```

## 3. Notfall

Zwei Knöpfe mit unterschiedlicher Wirkung. „Alle stoppen“ lässt offene Trades mit ihren Safety-Orders weiterlaufen, damit sie in einem fallenden Markt nicht ungeschützt sind. „Alle Börsen sofort trennen“ storniert alles und löscht die Schlüssel.

```mermaid
sequenceDiagram
    autonumber 31
    actor N as Nutzer
    participant A as Handy-App
    participant S as Terando-Server
    participant B as Börse (z. B. Binance)

    rect rgba(179,38,30,0.08)
    Note over N,B: Notfall
    alt Alle stoppen
        N->>A: Alle stoppen
        A->>S: Stopp (HTTPS)
        S->>S: Keine neuen Trades, offene behalten Ziel, Stopp, Safety-Orders
        S-->>A: Gestoppt ✓
    else Alle Börsen sofort trennen
        N->>A: Alle Börsen trennen
        A->>S: Trennen (HTTPS)
        S->>B: Offene Orders stornieren (HTTPS)
        S->>S: Alle Schlüssel löschen, Bots stoppen
        S-->>A: Getrennt ✓
    end
    end
```

Es gibt keinen Pfeil „Geld abheben“: Der Server hat dieses Recht nicht.

## Was der Bot kann und was nicht

| Kann | Kann nicht |
| --- | --- |
| Kurse und Kontostand lesen | Geld abheben oder auf ein anderes Konto überweisen |
| Kaufen und verkaufen, nur die gewählten Coins | Mit einem Schlüssel weiterhandeln, der plötzlich Auszahlungen erlaubt |
| Pro Order höchstens den eingestellten Betrag | Den Schlüssel wieder anzeigen (sichtbar sind nur die letzten 4 Zeichen) |
| Pro Coin und Konto nur einen offenen Bot-Trade halten | |

## Was schützt

- Der Schlüssel funktioniert nur vom Terando-Server aus (IP-Beschränkung). Ein gestohlener Schlüssel ist woanders nutzlos.
- Stündliche Prüfung der Schlüsselrechte.
- Feste Coin-Liste, Höchstbetrag, keine illiquiden Paare.
- 2FA, Sperre nach Fehlversuchen, Warnung bei neuem Gerät. Ein Passwortwechsel trennt alle Börsen.
- Jede Verbindung läuft über HTTPS, auch zur Börse. Schlüssel werden verschlüsselt gespeichert.

Die stündliche Prüfung kam nach einem externen Review dieses Datenflusses dazu. Vorher wurden die Rechte nur beim Verbinden geprüft.

📄 Dasselbe als PDF zum Ausdrucken: [terando-datenfluss.pdf](docs/terando-datenfluss.pdf)
