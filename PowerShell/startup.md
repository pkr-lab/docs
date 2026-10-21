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
1. Internetverbindung prüfen (5 Versuche, je 10 Sek.)
2. Bei kein Internet: WLAN-Meldebox anzeigen und beenden
3. Quelldateien von SharePoint/OneDrive-Sync nach `C:\Users\Public\10020115_WinScripts\Win11_C` kopieren
4. Skript-Kodierung prüfen (UTF-16LE / UTF-8 BOM)
5. Alle `.ps1`-Skripte in `Software\Scripte\` ausführen (außer `StartUp\`)

**Wichtige Konfiguration:**
```powershell
$Source = "C:\Users\DLRG-JugendAndernach\DLRG\...\Win11_C"   # Sync-Quelle
$DestinationRoot = "C:\Users\Public\10020115_WinScripts"       # Ziel
```

**Einrichten:** Über `Ausführung\WatchDogInstall.ps1` (als normaler Benutzer ausführen)

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

Die Meldungen aller Skripte sammelt [Debugging und Fehlersuche](debugging.md), die Logdateien liegen unter `C:\Users\Public\10020115_WinScripts\Logs\`. Der Watchdog protokolliert in eine eigene Datei je Lauf (`log_<datum>_<uhrzeit>.log`).

| Skript | Symptom | Ursache und Lösung |
|---|---|---|
| Watchdog | `Keine Internetverbindung nach 5 Versuchen.` | Prüfung gegen `https://www.google.com/generate_204` scheitert (Netz, Proxy, Firewall), Lauf endet mit Exit 1 |
| | `Quellpfad existiert nicht: <pfad>` | `$Source` zeigt ins Leere (OneDrive nicht synchron, anderer Benutzername) |
| | `Loeschen des Zielordners fehlgeschlagen` / `Kopiervorgang fehlgeschlagen` | Dateien gesperrt oder Quelle nicht lokal verfügbar (OneDrive „Dateien bei Bedarf“) |
| | `Message-Script nicht gefunden (…)` | das Meldungsskript heißt im Watchdog `10020115_Win_S_Messagebox.ps1`, im Repo liegt `10020115_Win_S_MessageWLAN.ps1` (TODO: prüfen) |
| | `Fertig: <Skript>`, aber keine Wirkung | der Watchdog wertet den Exit-Code der Kindskripte nicht aus, Fehler stehen im Log des Kindskripts ([Debugging](debugging.md#wichtige-verhaltensweisen-des-watchdogs)) |
| MessageWLAN | Box erscheint nie oder sofort | Zählweise: [Debugging](debugging.md#wlan-erinnerung-10020115_win_s_messagewlanps1) |
| Win_A_Install | `Verzeichnis konnte nicht erstellt werden: …` | Rechte unter `C:\Users\Public` |
| | `Registry-Pfad nicht gefunden: …`, `Keine klassischen Programme gefunden.` | Inventur liefert nichts, Registry prüfen |
