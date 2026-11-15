# StartUp – Skripte für Login-Initialisierung

Skripte in diesem Ordner werden **vom Watchdog-Task ausgeschlossen** und daher **nicht automatisch** bei jedem Login ausgeführt. Sie sind für spezielle Zwecke (Erst-Einrichtung, Login-Meldungen) gedacht.

## Dateien

| Skript | Zweck | Ausführung | Admin? |
|--------|-------|-----------|--------|
| `10020115_Win_A_Install.ps1` | Erst-Einrichtung: Verzeichnisse + Software-Inventur | Einmalig manuell | Nein |
| `10020115_Win_S_MessageWLAN.ps1` | WLAN-Erinnerungs-Popup nach X Tagen ohne WLAN | Per Scheduled Task | Nein |
| `10020115_Win_S_Watchdog.ps1` | **Hauptprozess:** Dateien synchronisieren + alle Skripte ausführen | Per Scheduled Task (Login) | Ja |

---

## `10020115_Win_S_Watchdog.ps1`

**Zweck:** Herzstück des Systems. Läuft bei jedem Benutzer-Login.

**Ablauf:**
1. `Logging.psm1` und `Common.psm1` aus dem Ordner über dem Watchdog laden. Die Module bleiben im Speicher, auch wenn Schritt 4 ihre Dateien löscht.
2. Sync-Quelle aus `WatchdogQuelle` in `Konfiguration.psd1` lesen ([Konfiguration](konfiguration.md#lokale-konfiguration-konfigurationpsd1)), ohne Wert Abbruch mit Exit 1
3. Internetverbindung prüfen (5 Versuche, je 10 Sek.), ohne Verbindung `10020115_Win_S_MessageWLAN.ps1` starten und beenden
4. Quelldateien von SharePoint/OneDrive-Sync nach `C:\Users\Public\10020115_WinScripts\Win11_C` kopieren
5. Alle `.ps1`-Skripte in Unterordnern von `Software\Scripte\` ausführen, außer in `StartUp\` und `Ausfuehrung\`. Skripte direkt in `Software\Scripte\` (z. B. `SSH_toMac.ps1`) startet der Watchdog nicht.
6. Exit-Code jedes Kindskripts auswerten: `0` ergibt `Fertig: <Skript>`, alles andere `Fehler in <Skript>: Exit-Code <n>`

Die Skripte liegen als UTF-8 mit BOM im Repo, eine Umwandlung der Kodierung zur Laufzeit gibt es nicht mehr.

**Einrichten:** Über `Ausfuehrung\WatchDogInstall.ps1` (als normaler Benutzer ausführen). Vorher muss `Win11_C` einmal vollständig nach `C:\Users\Public\10020115_WinScripts\Win11_C` kopiert sein, einschließlich `Software\Scripte\Konfiguration.psd1`.

---

## `10020115_Win_S_MessageWLAN.ps1`

**Zweck:** Zeigt eine Erinnerungs-Meldebox an, wenn der PC länger als N Tage nicht im WLAN war.

**Parameter (im Skript änderbar):**

| Variable | Standard | Beschreibung |
|----------|---------|--------------|
| `$threshold` | `20` | Tage ohne WLAN bis zur Meldung |
| `$counterFile` | `...\WLANCounter.txt` | Datei zum Speichern des Tages-Zählers |

**Was wird geändert:**
- Zähldatei: `C:\Users\Public\10020115_WinScripts\10020115_Win_R_WLANCounter.txt`
- Kein Systemeingriff; nach Bestätigung wird Zähler zurückgesetzt

---

## `10020115_Win_A_Install.ps1`

**Zweck:** Erste-Einrichtung eines neuen Geräts.

**Aktionen:**
- Erstellt Basisverzeichnisse (`Logs`, `Win11_C`)
- Erstellt Software-Inventur im Log

```powershell
# Einmalig auf neuem Gerät ausführen
.\10020115_Win_A_Install.ps1
```

---

## Fehlersuche

Die Meldungen aller Skripte sammelt [Debugging und Fehlersuche](debugging.md), die Logdateien liegen unter `C:\Users\Public\10020115_WinScripts\Logs\`. Der Watchdog protokolliert in eine eigene Datei je Lauf (`log_<datum>_<uhrzeit>.log`) und nutzt dafür `Logging.psm1`.

| Skript | Symptom | Ursache und Lösung |
|---|---|---|
| Watchdog | `Keine Internetverbindung nach 5 Versuchen.` | Prüfung gegen `https://www.google.com/generate_204` scheitert (Netz, Proxy, Firewall), Lauf endet mit Exit 1 |
| | `Abbruch: Konfigurationsdatei fehlt: …` / `Abbruch: Konfigurationsdatei … ohne Wert fuer: WatchdogQuelle` | `Konfiguration.psd1` fehlt im Zielordner oder ist unvollständig, siehe [Konfiguration](konfiguration.md#fehlersuche) |
| | `Quellpfad existiert nicht: <pfad>` | `WatchdogQuelle` zeigt ins Leere (OneDrive nicht synchron, anderer Benutzername) |
| | `Loeschen des Zielordners fehlgeschlagen` / `Kopiervorgang fehlgeschlagen` | Dateien gesperrt oder Quelle nicht lokal verfügbar (OneDrive „Dateien bei Bedarf“) |
| | `Message-Script nicht gefunden (…)` | `StartUp\10020115_Win_S_MessageWLAN.ps1` fehlt in Quelle und Ziel |
| | `Fehler in <Skript>: Exit-Code <n>` | Kindskript ist mit Fehler beendet, Details im Log des Kindskripts ([Debugging](debugging.md#wichtige-verhaltensweisen-des-watchdogs)) |
| MessageWLAN | Box erscheint nie oder sofort | Zählweise: [Debugging](debugging.md#wlan-erinnerung-10020115_win_s_messagewlanps1) |
| Win_A_Install | `Verzeichnis konnte nicht erstellt werden: …` | Rechte unter `C:\Users\Public` |
| | `Registry-Pfad nicht gefunden: …`, `Keine klassischen Programme gefunden.` | Inventur liefert nichts, Registry prüfen |
