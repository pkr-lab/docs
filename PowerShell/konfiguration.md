# Konfiguration

Werte, die je Einrichtung verschieden sind oder nicht ins öffentliche Repo gehören, stehen in der lokalen Datei `Konfiguration.psd1`. Alle übrigen Einstellungen sind Parameter oder Variablen am Anfang des jeweiligen Skripts.

## Lokale Konfiguration (`Konfiguration.psd1`)

Die Datei liegt neben `Logging.psm1` und `Common.psm1` im Wurzelordner der Skripte. In der Sync-Quelle ist das `Win11_C\Software\Scripte\Konfiguration.psd1`, der Watchdog kopiert sie bei jedem Lauf mit auf das Gerät. Im Repo ist sie per `.gitignore` ausgeschlossen. Als Vorlage dient `Konfiguration.example.psd1`, sie liegt auch im Release-ZIP:

```powershell
Copy-Item .\Konfiguration.example.psd1 .\Konfiguration.psd1
```

| Schlüssel | Genutzt von | Pflicht | Bedeutung |
|---|---|---|---|
| `WatchdogQuelle` | `StartUp/10020115_Win_S_Watchdog.ps1` | ja | lokaler Pfad zum synchronisierten Ordner `Win11_C` (OneDrive/SharePoint) |
| `SupportTelefon` | `Richtlinien/10020115_Win_R_SystemHilfe.ps1` | ja | Telefonnummer in den OEM-Angaben und im Anmeldehinweis |
| `RdpErlauben` | `Richtlinien/10020115_Win_R_Firewall.ps1` | nein, Standard `$false` | `$true` aktiviert die eingehende Firewall-Regel für Remotedesktop |
| `SshZielMac` | `SSH_toMac.ps1` | nur ohne `-TargetMac` | MAC-Adresse des SSH-Ziels (`XX-XX-XX-XX-XX-XX`) |
| `VmBenutzer` | `Ausfuehrung/DateienSendenVM.ps1` | nur ohne `-DestinationInVM` | Benutzer in der Test-VM, daraus wird `C:\Users\<VmBenutzer>\Desktop\PowerShell.zip` |

Gelesen wird die Datei mit `Get-Konfiguration` aus `Common.psm1`. Fehlt die Datei oder steht bei einem Pflichtwert nichts bzw. noch der Platzhalter `<…>`, brechen Watchdog und SystemHilfe mit einer `ERROR`-Meldung ab. Die Firewall läuft dann trotzdem durch und lässt RDP unverändert.

Der Watchdog liest seine Quelle aus der Kopie neben sich, also aus dem letzten Lauf oder der Ersteinrichtung. Danach löscht er den Zielordner und kopiert ihn neu. Eine geänderte `WatchdogQuelle` gilt deshalb erst ab dem übernächsten Lauf.

## Parameter und Variablen

| Skript | Parameter / Variable | Standard | Bedeutung |
|---|---|---|---|
| `StartUp/10020115_Win_S_Watchdog.ps1` | `$Source` | `WatchdogQuelle` aus `Konfiguration.psd1` | Sync-Quelle der Skripte |
| | `$DestinationRoot` | `C:\Users\Public\10020115_WinScripts` | lokales Zielverzeichnis |
| | `$MessageScriptRelative` | `Software\Scripte\StartUp\10020115_Win_S_MessageWLAN.ps1` | Meldungsskript bei fehlender Verbindung |
| | `$ExcludedFolders` | `StartUp`, `Ausfuehrung` | Ordner, deren Skripte der Watchdog nicht startet; Skripte direkt in `Software\Scripte` startet er ebenfalls nicht |
| | Internetprüfung | 5 Versuche, 10 s Pause, 10 s Timeout | Parameter `Retries`, `DelaySeconds`, `TimeoutSec` |
| `StartUp/10020115_Win_S_MessageWLAN.ps1` | `$threshold` | `20` | Tage bis zur WLAN-Erinnerung |
| | `$baseDir` | `C:\Users\Public\10020115_WinScripts` | Ablage der Zähldatei |
| `Ausfuehrung/WatchDogInstall.ps1` | `-TaskName`, `-ScriptPath` | `GPO-Watchdog`, Pfad zum Watchdog | Name der Aufgabe und Zielskript |
| `Ausfuehrung/DateienSendenVM.ps1` | `-VMName`, `-SourceZip`, `-DestinationInVM`, `-GpoName`, `-GpoBackupPath` | Test-VM, ZIP-Pfade, Ziel in der VM aus `VmBenutzer`, `C:\GPO-Backups` | Hyper-V-Ziel und GPO-Export |
| `Update/10020115_Win_UpdateScript.ps1` | `-RUN_IN_BACKGROUND`, `-LOGFILE` | aus | Hintergrundlauf und Logdatei |
| `InstallSoftware/10020115_Win_A_InstallBIOStools.ps1` | `-InstallerPath` | `LenovoBiosInstall\system_update.exe` | Lenovo-Installer; HP nutzt das Modul HPCMSL |
| `Programme/10020115_Win_doc_Word.ps1` | `-ModulePath`, `-TemplateSourcePath`, Log-Level | `..\Logging.psm1` neben dem Skript, Vorlagen unter `Win11_C` | Quelle der Word-Vorlagen |
| `Richtlinien/10020115_Win_R_SystemHilfe.ps1` | `$Manufacturer`, `$SupportHours`, `$SupportURL`, `$Model`, `$LogoPath`; `$SupportPhone` aus `SupportTelefon` | Werte für die OEM-Anzeige | Branding und Support-Angaben |
| `Richtlinien/10020115_Win_R_Personalisierung.ps1` | `$UserWallpaper` | `…\Bilder\Hintergrund\Desktop.png` | Desktop-Hintergrund |
| `Richtlinien/10020115_Win_R_Firewall.ps1` | `RdpErlauben` | `$false` | RDP-Regel aktivieren |
| `SSH_toMac.ps1` | `-TargetMac`, `-SshUser` | `SshZielMac` aus `Konfiguration.psd1`, aktueller Benutzer | Ziel für die SSH-Verbindung per MAC-Adresse |

## Logging (`Logging.psm1`)

Alle Skripte nutzen `Logging.psm1` für einheitliches Logging. Sie laden das Modul relativ zum eigenen Ordner: Skripte in Unterordnern mit `..\Logging.psm1`, der Watchdog genauso.

```powershell
Import-Module (Join-Path $PSScriptRoot '..\Logging.psm1')

Initialize-Logger -Level "INFO"               # Level: DEBUG | INFO | WARN | ERROR
Write-Log -Level INFO  -Message "Alles OK"
Write-Log -Level ERROR -Message "Etwas lief schief"
Close-Logger                                   # Optional: sauber schließen
```

**Log-Dateien:** `C:\Users\Public\10020115_WinScripts\Logs\log_YYYY-MM-DD.log`  
**Log-Rotation:** täglich (neuer Dateiname) und nach Größe (Standard: 5 MB → `.bak`), Funktion `Invoke-LogRotation`
**Kopfzeile:** Wechselt das schreibende Skript, steht `+++ <Skriptname> +++` im Log.

Der Zustand des Loggers (Pfad, Level, Konsolenausgabe) liegt in Skriptvariablen des Moduls (`$script:`), nicht mehr global.

## Gemeinsame Funktionen (`Common.psm1`)

| Funktion | Zweck |
|---|---|
| `Test-IsAdmin` | `$true`, wenn der Prozess als Administrator läuft |
| `Get-InstalledProgram` | klassische Programme aus den Uninstall-Schlüsseln (64 und 32 Bit) mit Name, Version, Publisher, InstallDate |
| `Initialize-PackageInfrastructure` | TLS 1.2, NuGet-Provider (AllUsers, sonst CurrentUser) und PSGallery als `Trusted` |
| `Get-Konfiguration` | liest `Konfiguration.psd1`, `-Pflichtwert` prüft Schlüssel, `-Optional` liefert ohne Datei eine leere Tabelle |

## Pfade

| Pfad | Inhalt |
|---|---|
| `C:\Users\Public\10020115_WinScripts\Win11_C\` | lokale Kopie der Skripte |
| `C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\Konfiguration.psd1` | lokale Konfiguration (Kopie aus der Sync-Quelle) |
| `C:\Users\Public\10020115_WinScripts\Logs\` | Logdateien |
| `C:\Users\Public\10020115_WinScripts\10020115_Win_R_WLANCounter.txt` | Zähldatei der WLAN-Erinnerung |


## Fehlersuche

Falsche Werte zeigen sich als Meldung im Log ([Debugging](debugging.md)).

| Meldung | Ursache | Lösung |
|---|---|---|
| `Konfigurationsdatei fehlt: <pfad> (Vorlage: Konfiguration.example.psd1)` | `Konfiguration.psd1` liegt nicht in der Sync-Quelle bzw. neben `Common.psm1` | Vorlage kopieren und ausfüllen |
| `Konfigurationsdatei <pfad> ohne Wert fuer: <schlüssel>` | Schlüssel fehlt, ist leer oder enthält noch den Platzhalter | Wert eintragen |
| `Import-Module … Logging.psm1` bzw. `Common.psm1` scheitert | Skript liegt nicht eine Ebene unter den Modulen (Struktur der Sync-Quelle weicht von `Software\Scripte\<Ordner>\` ab) | Struktur wie im Repo unter `powershell/` ablegen |
