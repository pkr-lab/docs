# Update – Windows- und Software-Updates

## Dateien

| Skript | Zweck | Als Admin? |
|--------|-------|-----------|
| `10020115_Win_UpdateScript.ps1` | Windows-Updates + Treiber + Winget (im Hintergrund) | Ja |

---

## `10020115_Win_UpdateScript.ps1`

**Zweck:** Vollautomatisches Patching: Windows-Updates (via PSWindowsUpdate), Treiberupdates und Software-Updates (via winget).

**Besonderheit – Hintergrundmodus:**  
Das Skript startet sich selbst als Hidden-Prozess neu, damit der aufrufende Terminal-Prozess (Watchdog) nicht blockiert wird.

**Ablauf:**
1. Erster Aufruf: startet sich selbst mit `-RUN_IN_BACKGROUND` im Hintergrund
2. Hintergrundprozess:
   - PSWindowsUpdate installieren (falls nötig)
   - Windows-Updates suchen, loggen, installieren
   - Treiberupdates aktivieren
   - Winget-Quellen aktualisieren + alle Pakete upgraden

**Abhängigkeiten:**
- `Logging.psm1`
- PowerShell-Modul `PSWindowsUpdate` (wird automatisch installiert)
- `winget` (Windows Package Manager, ab Windows 10 21H1)

**Logdateien:** `C:\Users\Public\10020115_WinScripts\Logs\update_YYYY-MM-DD_HH-mm-ss.log`

```powershell
# Manuell ausführen (als Administrator)
.\10020115_Win_UpdateScript.ps1
```

**Was wird am PC geändert:**
- Windows-Updates installiert (Sicherheit, Qualität)
- Treiberupdates installiert
- Software via winget aktualisiert
- Registry: `HKLM:\...\DriverSearching\SearchOrderConfig = 1`
- PSWindowsUpdate-Modul installiert (bei Erstlauf)

**Hinweis:** `-IgnoreReboot` – kein automatischer Neustart. Bitte manuell neu starten, falls Updates es erfordern.

---

## Fehlersuche

Die Meldungen aller Skripte sammelt [Debugging und Fehlersuche](debugging.md), die Logdateien liegen unter `C:\Users\Public\10020115_WinScripts\Logs\`. Das Update-Skript startet sich selbst im Hintergrund: Der erste Aufruf legt die Logdatei `update_<datum>_<uhrzeit>.log` an und startet einen versteckten Prozess mit `-RUN_IN_BACKGROUND`. Die Ausgabe steht daher in dieser Datei und nicht in der Konsole.

| Meldung oder Symptom | Ursache und Lösung |
|---|---|
| `LOGFILE-Parameter fehlt.` | `-RUN_IN_BACKGROUND` ohne `-LOGFILE` aufgerufen, das Skript ohne Schalter starten |
| `Installation von PSWindowsUpdate fehlgeschlagen: …` | kein Netz, PSGallery nicht erreichbar oder keine Rechte (`Install-Module PSWindowsUpdate -Force` als Administrator) |
| `Fehler beim Abrufen der Updates: …`, `Fehler bei der Update-Installation: …` | Windows-Update-Dienst gestört (`wuauserv`), `Get-WindowsUpdateLog` |
| `Treiberupdate-Einstellung konnte nicht gesetzt werden: …` | Registry-Recht, Warnung |
| `winget nicht gefunden – Software-Updates werden uebersprungen.` | App Installer fehlt |
| `Fehler bei Winget-Updates: …` | `winget upgrade` von Hand prüfen |
| Konsole bleibt leer | gewollt: Hintergrundmodus, Log lesen |
