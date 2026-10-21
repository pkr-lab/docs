# Konfiguration

Die Skripte haben keine Konfigurationsdatei. Einstellungen stehen als Parameter oder als Variablen am Anfang des jeweiligen Skripts.

## Parameter und Variablen

| Skript | Parameter / Variable | Standard | Bedeutung |
|---|---|---|---|
| `StartUp/10020115_Win_S_Watchdog.ps1` | `$Source` | lokaler OneDrive-Pfad `…\Win11_C` | Sync-Quelle der Skripte |
| | `$DestinationRoot` | `C:\Users\Public\10020115_WinScripts` | lokales Zielverzeichnis |
| | `$MessageScriptRelative` | `Software\Scripte\StartUp\10020115_Win_S_Messagebox.ps1` | Meldungsskript bei fehlender Verbindung |
| | Internetprüfung | 5 Versuche, 10 s Pause, 10 s Timeout | Parameter `Retries`, `DelaySeconds`, `TimeoutSec` |
| `StartUp/10020115_Win_S_MessageWLAN.ps1` | `$threshold` | `20` | Tage bis zur WLAN-Erinnerung |
| | `$baseDir` | `C:\Users\Public\10020115_WinScripts` | Ablage der Zähldatei |
| `Ausführung/WatchDogInstall.ps1` | `-TaskName`, `-ScriptPath` | `GPO-Watchdog`, Pfad zum Watchdog | Name der Aufgabe und Zielskript |
| `Ausführung/DateienSendenVM.ps1` | `-VMName`, `-SourceZip`, `-DestinationInVM`, `-GpoName`, `-GpoBackupPath` | Test-VM, ZIP-Pfade, `C:\GPO-Backups` | Hyper-V-Ziel und GPO-Export |
| `Update/10020115_Win_UpdateScript.ps1` | `-RUN_IN_BACKGROUND`, `-LOGFILE` | aus | Hintergrundlauf und Logdatei |
| `InstallSoftware/10020115_Win_A_InstallBIOStools.ps1` | `-InstallerPath` | `LenovoBiosInstall\system_update.exe` | Lenovo-Installer; HP nutzt das Modul HPCMSL |
| `Porgamme/10020115_Win_doc_Word.ps1` | `-ModulePath`, `-TemplateSourcePath`, Log-Level | Pfade unter `Win11_C` | Quelle der Word-Vorlagen |
| `Richtlinien/10020115_Win_R_SystemHilfe.ps1` | `$Manufacturer`, `$SupportHours`, `$SupportPhone`, `$SupportURL`, `$Model`, `$LogoPath` | Werte für die OEM-Anzeige | Branding und Support-Angaben |
| `Richtlinien/10020115_Win_R_Personalisierung.ps1` | `$UserWallpaper` | `…\Bilder\Hintergrund\Desktop.png` | Desktop-Hintergrund |
| `SSH_toMac.ps1` | `-TargetMac`, `-SshUser` | feste MAC, aktueller Benutzer | Ziel für die SSH-Verbindung per MAC-Adresse |

## Logging (`Logging.psm1`)

Alle Skripte nutzen `Logging.psm1` für konsistentes Logging.

```powershell
Import-Module "C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\Logging.psm1"

Initialize-Logger -Level "INFO"               # Level: DEBUG | INFO | WARN | ERROR
Write-Log -Level INFO  -Message "Alles OK"
Write-Log -Level ERROR -Message "Etwas lief schief"
Close-Logger                                   # Optional: sauber schließen
```

**Log-Dateien:** `C:\Users\Public\10020115_WinScripts\Logs\log_YYYY-MM-DD.log`  
**Log-Rotation:** Täglich (neuer Dateiname) + nach Größe (Standard: 5 MB → `.bak`)

## Pfade

| Pfad | Inhalt |
|---|---|
| `C:\Users\Public\10020115_WinScripts\Win11_C\` | lokale Kopie der Skripte |
| `C:\Users\Public\10020115_WinScripts\Logs\` | Logdateien |
| `C:\Users\Public\10020115_WinScripts\10020115_Win_R_WLANCounter.txt` | Zähldatei der WLAN-Erinnerung |


## Fehlersuche

Die Skripte haben keine Konfigurationsdatei, falsche Werte zeigen sich als Meldung im Log ([Debugging](debugging.md)). Häufig: feste Pfade (`C:\Users\Public\10020115_WinScripts\…`, `Software\Scripte\…`) stimmen nicht mit der Ablage der Sync-Quelle überein, `$Source` im Watchdog enthält einen Benutzernamen, oder `Logging.psm1` liegt nicht am erwarteten Pfad (`Import-Module … Logging.psm1` scheitert).
