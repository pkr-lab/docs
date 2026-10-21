# InstallSoftware – Hersteller-spezifische Tool-Installation

Skripte zur Installation von herstellerspezifischen Verwaltungswerkzeugen (BIOS, Firmware, Treiber).

## Dateien

| Skript | Zweck | Als Admin? |
|--------|-------|-----------|
| `10020115_Win_A_InstallBIOStools.ps1` | BIOS/Firmware-Tool je nach Gerätehersteller installieren | Empfohlen |

---

## `10020115_Win_A_InstallBIOStools.ps1`

**Zweck:** Erkennt den Gerätehersteller automatisch und installiert das passende Update-Tool.

| Hersteller | Tool | Quelle |
|-----------|------|--------|
| HP | HPCMSL (HP Client Management Script Library) | PowerShell Gallery |
| Lenovo | Lenovo System Update | Lokale `.exe` |
| Sonstige | Keine Aktion | – |

**Abhängigkeiten:**
- `Logging.psm1` (aus `Software\Scripte\`)
- Internetverbindung (HP) bzw. lokale Installer-Datei `LenovoBiosInstall\system_update.exe` (Lenovo)

**Ausführung:**
```powershell
# Wird automatisch vom Watchdog aufgerufen, oder manuell:
.\10020115_Win_A_InstallBIOStools.ps1
```

**Was wird am PC geändert:**
- HP: PowerShell-Modul `HPCMSL` installiert (aus PSGallery)
- Lenovo: `system_update.exe` installiert Lenovo System Update
- Registry: NuGet-PackageProvider, PSGallery als Trusted

**Rollback (HP):**
```powershell
Uninstall-Module -Name HPCMSL -AllVersions -Force
```

---

## Fehlersuche

Die Meldungen aller Skripte sammelt [Debugging und Fehlersuche](debugging.md), die Logdateien liegen unter `C:\Users\Public\10020115_WinScripts\Logs\`.

| Meldung | Ursache und Lösung |
|---|---|
| `Lenovo Installer nicht gefunden: <pfad>` und `Lenovo Installer fehlt. Bitte von https://support.lenovo.com herunterladen.` | `system_update.exe` liegt nicht am Pfad (`-InstallerPath`) |
| `Unbekannter Hersteller '<name>' – keine Installation durchgefuehrt.` | weder HP noch Lenovo, normal bei anderen Geräten |
| `Kritischer Fehler: …` | Installation scheiterte, Detail im Log, Modul `HPCMSL` (HP) per `Install-Module` prüfen |
