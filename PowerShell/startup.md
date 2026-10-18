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
