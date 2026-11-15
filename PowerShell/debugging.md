# Debugging und Fehlersuche

Code: [pkr-lab/Scripts](https://github.com/pkr-lab/Scripts), vor allem [powershell/Logging.psm1](https://github.com/pkr-lab/Scripts/blob/master/powershell/Logging.psm1), [powershell/Common.psm1](https://github.com/pkr-lab/Scripts/blob/master/powershell/Common.psm1) und [powershell/StartUp/10020115_Win_S_Watchdog.ps1](https://github.com/pkr-lab/Scripts/blob/master/powershell/StartUp/10020115_Win_S_Watchdog.ps1). Die Pfade in dieser Doku sind relativ zu `powershell/` im Repo bzw. zu `Software\Scripte\` auf dem Gerät.

Das projektübergreifende Kapitel ist die [zentrale Fehlersuche](../capulus/capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

---

## Wo Logs entstehen

| Quelle | Pfad | Inhalt |
|---|---|---|
| Logging-Modul (`Initialize-Logger`) | `C:\Users\Public\10020115_WinScripts\Logs\log_JJJJ-MM-TT.log` | alle Skripte, die `Logging.psm1` nutzen, mit Kopfzeile `+++ <Skriptname> +++` bei jedem Skriptwechsel |
| Update-Skript | `...\Logs\update_JJJJ-MM-TT_HH-mm-ss.log` | der Hintergrundlauf der Updates, eigene Datei je Lauf (Größenlimit 10 MB) |
| Watchdog | `...\Logs\log_JJJJ-MM-TT_HH-mm-ss.log` | der Watchdog nutzt ebenfalls `Logging.psm1`, aber mit Zeitstempel im Dateinamen, eine Datei je Lauf |
| Rotation | `<Logdatei>.<Zeitstempel>.bak` | ab 5 MB (Standard) wird die Datei verschoben und neu begonnen |
| Konsole | farbig: ERROR rot, WARN gelb, INFO cyan, DEBUG grau | nur bei sichtbarem Fenster. Der Watchdog-Task startet versteckt (`-WindowStyle Hidden`) |
| Windows | Aufgabenplanung → Aufgabe `GPO-Watchdog` → Verlauf | zeigt nur, ob der Start gelang, nicht den Inhalt |

Format einer Zeile: `[JJJJ-MM-TT HH:mm:ss] [LEVEL] Text`. Log-Level: `DEBUG`, `INFO` (Standard), `WARN`, `ERROR`. Man setzt es mit `Initialize-Logger -Level "DEBUG"` am Anfang eines Skripts (weder `Konfiguration.psd1` noch eine Umgebungsvariable steuern es). `Get-LogConfig` zeigt die aktuellen Einstellungen.

---

## Lokal debuggen

```powershell
# Einzelnes Skript sichtbar und mit Debug-Log starten (Administrator-PowerShell für Richtlinien)
powershell.exe -ExecutionPolicy Bypass -File .\Richtlinien\10020115_Win_R_Firewall.ps1

# Logging-Modul isoliert prüfen
Import-Module .\Logging.psm1 -Force
Initialize-Logger -Level DEBUG
Write-Log -Level INFO -Message "Test"
Get-LogConfig

# Letzte Fehler der Logs
Select-String -Path C:\Users\Public\10020115_WinScripts\Logs\*.log -Pattern '\[ERROR\]|\[WARN\]' | Select-Object -Last 30

# Geplante Aufgabe prüfen
Get-ScheduledTask -TaskName GPO-Watchdog | Get-ScheduledTaskInfo
Get-ItemProperty HKCU:\Software\Microsoft\Windows\CurrentVersion\Run -Name GPOWatchdogStarter
```

Der PowerShell-ISE-Debugger oder `Set-PSBreakpoint -Script <Datei> -Line <n>` genügt für den Einzelschritt. Die Skripte laden `Logging.psm1` und `Common.psm1` relativ zu ihrem eigenen Ordner (`$PSScriptRoot\..`). Zum Debuggen genügt deshalb eine Kopie des ganzen `powershell/`-Ordners an beliebiger Stelle, mit `Konfiguration.psd1` für die Skripte, die sie brauchen.

---

## Wichtige Verhaltensweisen des Watchdogs

- **Exit-Code der Kindskripte.** Der Watchdog startet jedes Skript mit `Start-Process … -Wait -PassThru` in einem eigenen Prozess und wertet den Exit-Code aus. `0` ergibt `Fertig: <Skript>`, alles andere `Fehler in <Skript>: Exit-Code <n>`, auch bei einem `throw` (zum Beispiel „Administratorrechte erforderlich.“). Die Ursache steht im Log des Kindskripts (Kopfzeile `+++ <Skript> +++`), oder gar nicht, wenn das Skript vor dem Initialisieren des Loggers scheitert.
- **Zielordner wird vor jedem Lauf gelöscht.** Der Watchdog löscht `C:\Users\Public\10020115_WinScripts\Win11_C` vollständig und kopiert die Quelle neu. Lokale Änderungen an den Skripten gehen verloren. Zum Debuggen eine Kopie anlegen.
- **Internetprüfung:** Abruf von `https://www.google.com/generate_204`, 5 Versuche, 10 Sekunden Pause, 10 Sekunden Timeout. Ohne Verbindung beendet der Watchdog mit Exit-Code 1, nachdem er das Meldungsskript `StartUp\10020115_Win_S_MessageWLAN.ps1` ausgeführt hat (zuerst aus der Quelle, sonst aus dem Ziel).
- **Konfiguration:** Die Sync-Quelle kommt aus `WatchdogQuelle` in `Konfiguration.psd1` neben dem Watchdog ([Konfiguration](konfiguration.md#lokale-konfiguration-konfigurationpsd1)). Ohne die Datei bricht der Watchdog vor dem Kopieren ab.
- **Kodierung:** Die Skripte liegen als UTF-8 mit BOM im Repo (`.editorconfig`, Prüfung `PSUseBOMForUnicodeEncodedFile` in der CI). Windows PowerShell 5.1 liest sie damit richtig, eine Umwandlung zur Laufzeit gibt es nicht. Umlaut- oder Parsefehler nach dem Sync bedeuten, dass eine Datei ohne BOM in der Quelle liegt.
- **Ausführungsreihenfolge:** alle `.ps1` in Unterordnern von `Software\Scripte` rekursiv, außer unter `StartUp` und `Ausfuehrung`, in der Reihenfolge, in der das Dateisystem sie liefert. Skripte direkt in `Software\Scripte` (z. B. `SSH_toMac.ps1`) startet der Watchdog nicht. Es gibt keine Abhängigkeiten zwischen den Skripten.
- **Rechte:** Der Task läuft mit `-RunLevel Highest` für den eingerichteten Benutzer. Die Richtlinien-Skripte prüfen auf Administrator und brechen sonst ab.

---

## Fehlermeldungen

### Watchdog und Einrichtung

| Meldung | Ursache | Lösung |
|---|---|---|
| `Keine Internetverbindung (Versuch i/5): <detail>` / `Keine Internetverbindung nach 5 Versuchen.` | kein Netz oder `www.google.com` blockiert (Proxy, Firewall) | Netz und Proxy prüfen. Der Lauf endet mit Exit-Code 1 |
| `Abbruch: Keine Internetverbindung. Versuche Message-Script auszuführen.` | Folge der obigen Meldung | siehe oben |
| `Abbruch: Konfigurationsdatei fehlt: <pfad> …` / `Abbruch: Konfigurationsdatei <pfad> ohne Wert fuer: WatchdogQuelle` | `Konfiguration.psd1` fehlt neben dem Watchdog oder ist unvollständig | Datei in der Sync-Quelle und einmalig im Ziel anlegen, der Lauf endet mit Exit-Code 1 |
| `Message-Script nicht gefunden (Quelle: …, Ziel: …).` | Meldungsskript fehlt in Quelle und Ziel | Datei bereitstellen oder Pfad `$MessageScriptRelative` anpassen |
| `Message-Script beendet mit Exit-Code <n>: <pfad>` / `Ausführen des Message-Scripts fehlgeschlagen: <detail>` | Meldungsskript ist mit Fehler beendet bzw. ließ sich nicht starten | Skript einzeln starten |
| `Quellpfad existiert nicht: <pfad>` | `WatchdogQuelle` zeigt ins Leere (OneDrive nicht synchronisiert, anderer Benutzername) | Sync-Ordner und `WatchdogQuelle` prüfen, der Lauf endet mit Exit-Code 1 |
| `Loeschen des Zielordners fehlgeschlagen: <detail>` | Dateien im Ziel gesperrt (Skript läuft noch, Virenscanner) | laufende Skripte beenden, erneut starten |
| `Erstellen des Zielordners fehlgeschlagen: <detail>` | keine Rechte unter `C:\Users\Public` | Rechte prüfen |
| `Kopiervorgang fehlgeschlagen: <detail>` | Quelle nicht vollständig lokal verfügbar (OneDrive „Dateien bei Bedarf“), Pfad zu lang | Ordner in OneDrive „Immer auf diesem Gerät behalten“ |
| `Attrib entfernen fehlgeschlagen: …` / `Fehler beim Entfernen von Attributen: …` / `Attribute setzen fehlgeschlagen: …` | einzelne Datei lässt sich nicht umstellen | meist harmlos (Warnung) |
| `ScriptRoot existiert nicht: <pfad>` | Kopie war nicht vollständig | Kopierfehler weiter oben im Log suchen |
| `Keine Skripte gefunden (nach Ausschluss von Wurzelordner, StartUp, Ausfuehrung).` | Quelle enthält keine `.ps1` in den übrigen Unterordnern | Quelle prüfen |
| `Fehler in <Skript>: Exit-Code <n>` | Kindskript ist mit Fehler beendet | Log des Kindskripts lesen |
| `Fehler in <Skript>: <detail>` | Start des Kindprozesses scheiterte (nicht ein Fehler im Skript selbst) | `powershell.exe` im Pfad? |
| `Kein interaktiver Benutzer erkannt. Bitte PowerShell normal (nicht als Administrator) öffnen.` (`WatchDogInstall.ps1`, Exit 1) | kein angemeldeter Benutzer (zum Beispiel per Remote ohne Sitzung) | im Benutzerkontext starten |
| `Dieses Skript läuft als Administrator. Der Run-Key wird im Admin-Profil gesetzt …` (Warnung) | als Administrator gestartet | ohne „Als Administrator“ ausführen |
| `Watchdog-Skript nicht gefunden: <pfad>` (Warnung) | Standardpfad fehlt noch (erste Einrichtung) | Skripte zuerst kopieren oder `-ScriptPath` setzen |
| `Fehler beim Registrieren der Aufgabe: <detail>` (Exit 2) | keine Berechtigung für die Aufgabenplanung | Rechte prüfen |

### WLAN-Erinnerung (`10020115_Win_S_MessageWLAN.ps1`)

| Symptom | Ursache | Lösung |
|---|---|---|
| Meldebox erscheint nie | Das Skript zählt, an wie vielen Kalendertagen es **gelaufen** ist (einmal pro Tag, Zähldatei `10020115_Win_R_WLANCounter.txt` mit Inhalt `<zähler>;<datum>`) und zeigt die Box ab `$threshold` (20). Es prüft selbst keine WLAN-Verbindung | Der Watchdog ruft es nur ohne Internetverbindung auf. Zähldatei lesen |
| Meldebox erscheint sofort | Zähler steht schon über der Schwelle | Zähldatei auf `0;<heute>` setzen |
| Zähler wird nicht zurückgesetzt | Fenster nicht mit „OK“ geschlossen | Meldebox bestätigen: Dann setzt das Skript `0;<heute>` |
| Zähldatei beschädigt | unlesbarer Inhalt | das Skript fällt auf Zähler 0 zurück und schreibt die Datei neu |

### Richtlinien (Administrator nötig)

| Meldung | Skript | Ursache | Lösung |
|---|---|---|---|
| `Administratorrechte erforderlich.` (Log `ERROR`, danach `throw`) | Passwort, WinInstallStore, SoftwareInstallUninstall, Personalisierung, SystemHilfe, Firewall | nicht als Administrator gestartet | in einer Administrator-PowerShell starten, oder der Task läuft nicht erhöht |
| `Fehler beim Erstellen des Registry-Pfads: …` / `Fehler beim Setzen der PIN-Komplexitaet: …` | Passwort | Registry-Schreibzugriff abgelehnt | Rechte und Gruppenrichtlinien prüfen |
| `Fehler beim Deaktivieren des Stores: …` | WinInstallStore | HKLM-Schreibzugriff | wie oben |
| `Fehler beim Entfernen von SRP: …` / `Konnte nicht entfernen: <name> - …` | SoftwareInstallUninstall | Registry-Schlüssel gesperrt | manuell entfernen |
| `Benutzer-Hintergrund nicht gefunden: <pfad>` / `Sperrbildschirmbild nicht gefunden: <pfad>` | Personalisierung | Bilddatei fehlt | `$UserWallpaper` bzw. `$LockImageScreen` prüfen |
| `Fehler beim Setzen des Benutzer-Hintergrunds: …` / `Fehler beim Setzen der ADMX-Richtlinien: …` | Personalisierung | Rechte oder Pfad | Details lesen |
| `OEM-Logo nicht gefunden (wird uebersprungen): <pfad>` | SystemHilfe | `$LogoPath` falsch | Pfad prüfen, Branding läuft ohne Logo weiter |
| `Fehler beim OEM-Branding: …` / `Fehler beim Setzen der Legal Notice: …` / `Computerbeschreibung konnte nicht gesetzt werden: …` | SystemHilfe | Registry-Rechte | wie oben |
| `Fehler beim Aktivieren der Firewall: …` / `Fehler beim Setzen der Standardregeln: …` | Firewall | `Set-NetFirewallProfile` abgelehnt | Rechte, Fremd-Firewall prüfen |
| `RDP-Regel wird nicht aktiviert (RdpErlauben in Konfiguration.psd1 nicht gesetzt).` | Firewall | `RdpErlauben` fehlt oder ist `$false` | gewollt; für RDP `RdpErlauben = $true` setzen. Eine früher aktivierte Regel bleibt aktiv |
| `Konfiguration nicht lesbar, RDP bleibt unveraendert: …` | Firewall | `Konfiguration.psd1` fehlerhaft | Datei prüfen |
| `Konfiguration nicht verwendbar: …` (danach `throw`) | SystemHilfe | `SupportTelefon` fehlt in `Konfiguration.psd1` | Wert eintragen |
| `gpupdate beendet mit Exit-Code <n>` | Personalisierung | `gpupdate` scheiterte | `gpupdate /target:computer /force` von Hand, Ereignisanzeige |
| `Firewall-Regel nicht gefunden (uebersprungen): <regel>` / `Fehler bei Regel '<regel>': …` / `Public-Profil-Haertung teilweise fehlgeschlagen: …` / `Status-Abfrage fehlgeschlagen: …` | Firewall | Regel existiert in dieser Windows-Version nicht | Warnung, keine Aktion nötig |

### Update, Programme, Software

| Meldung | Skript | Ursache | Lösung |
|---|---|---|---|
| `LOGFILE-Parameter fehlt.` | Update (Hintergrundmodus) | Skript mit `-RUN_IN_BACKGROUND` ohne `-LOGFILE` gestartet | ohne Schalter starten, die beiden Parameter setzt das Skript selbst |
| `Import-Module … Logging.psm1` bzw. `Common.psm1` schlägt fehl | mehrere | Modul liegt nicht eine Ebene über dem Skript (Struktur der Quelle weicht ab) | Struktur `Software\Scripte\<Ordner>\<Skript>` herstellen |
| `Installation von PSWindowsUpdate fehlgeschlagen: …` | Update | kein Netz, PSGallery nicht erreichbar, keine Rechte | `Install-Module PSWindowsUpdate -Force` als Administrator |
| `Fehler beim Abrufen der Updates: …` / `Fehler bei der Update-Installation: …` | Update | Windows-Update-Dienst gestört | `Get-WindowsUpdateLog`, Dienst `wuauserv` prüfen |
| `Treiberupdate-Einstellung konnte nicht gesetzt werden: …` | Update | Registry-Recht | Warnung |
| `winget nicht gefunden – Software-Updates werden uebersprungen.` | Update | App-Installer fehlt | Microsoft „App Installer“ installieren |
| `winget source update beendet mit Exit-Code <n>.` / `Winget-Updates beendet mit Exit-Code <n>.` | Update | winget meldet einen Fehler. stderr-Ausgaben brechen den Abschnitt nicht mehr ab | Zeilen `winget: …` im Log lesen, `winget upgrade` von Hand |
| `Fehler bei Winget-Updates: …` | Update | winget ließ sich nicht starten | `winget upgrade` von Hand |
| `Keine klassischen Programme gefunden.` / `Fehler beim Auslesen klassischer Programme: …` / `Fehler beim Auslesen der UWP-Apps: …` | Log_Software, Win_A_Install | Registry bzw. AppX-Abfrage lieferte nichts | Warnung bzw. Details lesen |
| `Verzeichnis konnte nicht erstellt werden: <pfad> - …` | Win_A_Install | Pfad oder Rechte | Details lesen |
| `Logging-Modul nicht gefunden: <pfad>` / `Vorlagenordner wurde nicht gefunden: <pfad>` + `Vorlagenordner fehlt.` / `Keine .dotx Dateien im Ordner gefunden: <pfad>` + `Keine Vorlagen vorhanden.` | doc_Word | Modul oder Quelle fehlt | `-ModulePath`, `-TemplateSourcePath` prüfen |
| `Fehler beim Kopieren von <vorlage>: …` / `Konnte Registry für Version <v> nicht setzen: …` / `Konnte Vorlage nicht in Word öffnen: …` / `Zielvorlage nicht gefunden (wird übersprungen): …` / `Fehler beim Starten von Word COM: …` / `Fehler im Script: …` | doc_Word | Word nicht installiert oder gesperrt | Details lesen, Word schließen |
| `Lenovo Installer nicht gefunden: <pfad>` + `Lenovo Installer fehlt. Bitte von https://support.lenovo.com herunterladen.` | InstallBIOStools | `system_update.exe` nicht am Pfad | Installer bereitstellen oder `-InstallerPath` setzen |
| `Unbekannter Hersteller '<name>' – keine Installation durchgefuehrt.` | InstallBIOStools | Hersteller weder HP noch Lenovo | normal bei anderen Geräten |
| `Kritischer Fehler: …` | InstallBIOStools | Installation scheiterte | Details lesen |

### Hilfsskripte

| Meldung | Skript | Ursache | Lösung |
|---|---|---|---|
| `Quelldatei nicht gefunden: <pfad>` (Warnung) | DateienSendenVM | ZIP-Pfad falsch | `-SourceZip` setzen |
| `Ungültige Eingabe.` | DateienSendenVM | Menüeingabe unbekannt | gültige Nummer wählen |
| Fehler von `Copy-VMFile` / `Backup-GPO` / `Import-GPO` | DateienSendenVM | Hyper-V Guest Services nicht aktiv, Modul GroupPolicy fehlt, kein Admin | Gastdienste in der VM aktivieren, RSAT installieren |
| `FEHLER: Keine aktive Netzwerkverbindung gefunden.` (Exit 1) | SSH_toMac | keine IPv4-Adresse außer Loopback und 169.254.x | Netz prüfen |
| `FEHLER: Konfigurationsdatei fehlt: …` / `FEHLER: Konfigurationsdatei … ohne Wert fuer: SshZielMac` (Exit 1) | SSH_toMac | ohne `-TargetMac` gestartet und keine `SshZielMac` in `Konfiguration.psd1` | `-TargetMac` übergeben oder Wert eintragen |
| `Konfigurationsdatei fehlt: …` / `… ohne Wert fuer: VmBenutzer` | DateienSendenVM (Aktion 1) | ohne `-DestinationInVM` gestartet und kein `VmBenutzer` | `-DestinationInVM` übergeben oder Wert eintragen |
| `FEHLER: Geraet mit MAC <mac> nicht gefunden.` (Exit 1) | SSH_toMac | Gerät nicht im lokalen Subnetz oder aus | Gerät einschalten, `-TargetMac` im Format `XX-XX-XX-XX-XX-XX`, `nmap` installieren für gründlicheren Scan |

---

## Checkliste „Skript wirkt nicht“

1. **Läuft der Watchdog?** Aufgabenplanung → `GPO-Watchdog` → letzte Ausführung. Run-Key `GPOWatchdogStarter` vorhanden (`Get-ItemProperty …`)?
2. **Aktuelles Watchdog-Log lesen:** neueste Datei unter `C:\Users\Public\10020115_WinScripts\Logs\`. Steht dort `Keine Internetverbindung`, `Quellpfad existiert nicht` oder `Kopiervorgang fehlgeschlagen`? Dann ist der Sync das Problem.
3. **Skript im Ziel vorhanden?** `C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\...`. Skripte unter `StartUp` und `Ausfuehrung` und direkt in `Software\Scripte` führt der Watchdog nie aus.
4. **Log des Kindskripts:** Suche nach `+++ <Skriptname> +++` in den Logs und danach nach `[ERROR]`.
5. **Rechte:** Meldung `Administratorrechte erforderlich.`? Skript von Hand in einer Administrator-PowerShell starten.
6. **Einzeln starten:** `powershell.exe -ExecutionPolicy Bypass -File <skript>` und die Ausgabe lesen.
7. **Module und Konfiguration vorhanden:** `Test-Path C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\Logging.psm1`, dasselbe für `Common.psm1` und `Konfiguration.psd1`.
8. **Wirkung prüfen:** Richtlinien setzen Registry-Werte, siehe [Rollback-Tabelle in der Nutzung](nutzung.md#rollback) für die betroffenen Schlüssel. Einige Wirkungen (OEM-Branding, Store) werden erst nach einem Neustart sichtbar.

---

## Typische Probleme zwischen den Teilen

| Symptom | Ursache | Lösung |
|---|---|---|
| Nach dem Verschieben des Repos laufen die Skripte nicht | Pfade wie `C:\Users\Public\10020115_WinScripts\…` (Logs, Bilder, Vorlagen) stehen weiter fest in einzelnen Skripten, nur die Module werden relativ geladen | Struktur der Sync-Quelle beibehalten ([Konfiguration](konfiguration.md)) |
| Sync-Quelle für einen anderen Benutzer | `WatchdogQuelle` enthält den Benutzernamen | `Konfiguration.psd1` je Sync-Quelle anpassen |
| Skripte im Repo anders sortiert als in der Sync-Quelle | Repo-Struktur (`powershell/`) ≠ erwarteter Pfad `Software\Scripte\…` | Quelle muss die Struktur `Software\Scripte\<Ordner>` haben |
| Watchdog läuft, aber keine Wirkung | Kindskript scheiterte (`Fehler in <Skript>: Exit-Code <n>`) oder liegt in einem ausgeschlossenen Ordner | Logs der Kindskripte lesen |
