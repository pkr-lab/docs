# Installation

## Voraussetzungen

| Anforderung | Details |
|------------|---------|
| Windows | Windows 10 oder Windows 11 (Client) |
| PowerShell | 5.1 oder 7.x |
| Rechte | Lokale Administratorrechte für Systemeingriffe |
| Internet | Für Updates und Modul-Installation (PSWindowsUpdate, HPCMSL) |
| OneDrive/SharePoint | Sync-Quelle für den Watchdog (`WatchdogQuelle` in `Konfiguration.psd1`) |

## Setup

1. Den Inhalt von `powershell/` (bzw. des Release-ZIPs) in die Sync-Quelle legen: SharePoint/OneDrive-Ordner `Win11_C`, darin `Software\Scripte\`.
2. Dort `Konfiguration.example.psd1` nach `Konfiguration.psd1` kopieren und ausfüllen ([Konfiguration](konfiguration.md#lokale-konfiguration-konfigurationpsd1)). Die Datei ist nicht im Repo und nicht im ZIP.
3. Auf dem Client `Win11_C` einmalig nach `C:\Users\Public\10020115_WinScripts\Win11_C` kopieren. Der Watchdog liest seine Quelle aus der Kopie von `Konfiguration.psd1` neben sich.
4. Einmalig als normaler Benutzer den Watchdog einrichten:

```powershell
powershell.exe -ExecutionPolicy Bypass -File .\Ausfuehrung\WatchDogInstall.ps1
```

Das Skript legt die geplante Aufgabe `GPO-Watchdog` an und trägt den Run-Key `GPOWatchdogStarter` ein. Ab der nächsten Anmeldung kopiert der Watchdog die Skripte nach `C:\Users\Public\10020115_WinScripts\Win11_C\` und führt sie aus.

Die Erst-Einrichtung eines Clients erledigt `StartUp\10020115_Win_A_Install.ps1`, siehe [startup.md](startup.md).


## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Watchdog startet nach der Einrichtung nicht | als Administrator eingerichtet (Run-Key im falschen Profil) | `WatchDogInstall.ps1` als normaler Benutzer wiederholen, [Ausführung: Fehlersuche](ausfuehrung.md#fehlersuche) |
| Skripte werden blockiert | Execution Policy | `-ExecutionPolicy Bypass` (nur für den Prozess) |
| Nichts wird kopiert | Sync-Quelle nicht vorhanden oder nicht lokal verfügbar | [Debugging](debugging.md#fehlermeldungen) |
| `Abbruch: Konfigurationsdatei fehlt: …` im Watchdog-Log | Schritt 2 oder 3 fehlt | `Konfiguration.psd1` anlegen und ins Ziel kopieren |
