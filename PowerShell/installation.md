# Installation

## Voraussetzungen

| Anforderung | Details |
|------------|---------|
| Windows | Windows 10 oder Windows 11 (Client) |
| PowerShell | 5.1 oder 7.x |
| Rechte | Lokale Administratorrechte für Systemeingriffe |
| Internet | Für Updates und Modul-Installation (PSWindowsUpdate, HPCMSL) |
| OneDrive/SharePoint | Sync-Quelle für den Watchdog (im Skript konfigurierbar) |

## Setup

1. Die Skripte in die Sync-Quelle legen (SharePoint/OneDrive-Ordner `Win11_C`, im Watchdog als `$Source` gesetzt).
2. Auf dem Client einmalig als normaler Benutzer den Watchdog einrichten:

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\Ausführung\WatchDogInstall.ps1
```

Das Skript legt die geplante Aufgabe `GPO-Watchdog` an und trägt den Run-Key `GPOWatchdogStarter` ein. Ab der nächsten Anmeldung kopiert der Watchdog die Skripte nach `C:\Users\Public\10020115_WinScripts\Win11_C\` und führt sie aus.

Die Erst-Einrichtung eines Clients erledigt `StartUp\10020115_Win_A_Install.ps1`, siehe [startup.md](startup.md).
