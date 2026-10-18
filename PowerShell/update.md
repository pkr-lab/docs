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
