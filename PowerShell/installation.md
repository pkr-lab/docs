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


## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Watchdog startet nach der Einrichtung nicht | als Administrator eingerichtet (Run-Key im falschen Profil) | `WatchDogInstall.ps1` als normaler Benutzer wiederholen, [Ausführung: Fehlersuche](ausfuehrung.md#fehlersuche) |
| Skripte werden blockiert | Execution Policy | `-ExecutionPolicy Bypass` (nur für den Prozess) |
| Nichts wird kopiert | Sync-Quelle nicht vorhanden oder nicht lokal verfügbar | [Debugging](debugging.md#fehlermeldungen) |
