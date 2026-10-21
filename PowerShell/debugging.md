# Debugging und Fehlersuche

Code: [pkr-lab/PowerShell](https://github.com/pkr-lab/PowerShell), vor allem [Logging.psm1](https://github.com/pkr-lab/PowerShell/blob/main/Logging.psm1) und [StartUp/10020115_Win_S_Watchdog.ps1](https://github.com/pkr-lab/PowerShell/blob/main/StartUp/10020115_Win_S_Watchdog.ps1)

Hinweis zur Ablage: Im lokalen Arbeitsverzeichnis (`~/gitlab/powershell-bash/scripte`) liegen die Skripte inzwischen unter `powershell/` und das neue Skript unter `bash/git-pull-push/`. Im Git-Index von `pkr-lab/PowerShell` stehen sie noch im Wurzelverzeichnis (Stand der Prüfung 2026-10-21, die Umstellung war nicht committet). Die Dateipfade dieser Doku folgen der bisherigen Struktur, TODO: prüfen, ob sie nach dem Commit der Umstellung mit dem Präfix `powershell/` zu aktualisieren sind.

Das projektübergreifende Kapitel ist die [zentrale Fehlersuche](../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

---

## Wo Logs entstehen

| Quelle | Pfad | Inhalt |
|---|---|---|
| Logging-Modul (`Initialize-Logger`) | `C:\Users\Public\10020115_WinScripts\Logs\log_JJJJ-MM-TT.log` | alle Skripte, die `Logging.psm1` nutzen, mit Kopfzeile `+++ <Skriptname> +++` bei jedem Skriptwechsel |
| Update-Skript | `...\Logs\update_JJJJ-MM-TT_HH-mm-ss.log` | der Hintergrundlauf der Updates, eigene Datei je Lauf (Größenlimit 10 MB) |
| Watchdog | `...\Logs\log_JJJJ-MM-TT_HH-mm-ss.log` | der Watchdog hat einen eigenen, integrierten Logger mit Zeitstempel im Dateinamen, eine Datei je Lauf |
| Rotation | `<Logdatei>.<Zeitstempel>.bak` | ab 5 MB (Standard) wird die Datei verschoben und neu begonnen |
| Konsole | farbig: ERROR rot, WARN gelb, INFO cyan, DEBUG grau | nur bei sichtbarem Fenster. Der Watchdog-Task startet versteckt (`-WindowStyle Hidden`) |
| Windows | Aufgabenplanung → Aufgabe `GPO-Watchdog` → Verlauf | zeigt nur, ob der Start gelang, nicht den Inhalt |

Format einer Zeile: `[JJJJ-MM-TT HH:mm:ss] [LEVEL] Text`. Log-Level: `DEBUG`, `INFO` (Standard), `WARN`, `ERROR`. Man setzt es mit `Initialize-Logger -Level "DEBUG"` am Anfang eines Skripts (es gibt keine Konfigurationsdatei und keine Umgebungsvariable dafür). `Get-LogConfig` zeigt die aktuellen Einstellungen.

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

Der PowerShell-ISE-Debugger oder `Set-PSBreakpoint -Script <Datei> -Line <n>` genügt für den Einzelschritt. Die Skripte brauchen dafür dasselbe Modul `Logging.psm1` am Pfad `C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\Logging.psm1` (fest eingetragen in mehreren Skripten). Fehlt dieser Pfad, scheitert der Import (siehe unten).

---

## Wichtige Verhaltensweisen des Watchdogs

- **Fehler in Kindskripten bleiben unsichtbar.** Der Watchdog startet jedes Skript mit `Start-Process … -Wait` in einem eigenen Prozess und wertet den Exit-Code nicht aus. Auch ein Skript, das mit `throw` abbricht (zum Beispiel „Administratorrechte erforderlich.“), erscheint im Watchdog-Log als `Fertig: <Skript>`. Die Ursache steht nur im Log des Kindskripts (am Dateinamen in der Kopfzeile `+++ <Skript> +++` erkennbar) oder gar nicht, wenn es vor dem Initialisieren des Loggers scheitert. Wer Fehler sucht, sucht daher in den Dateien unter `Logs\` nach `[ERROR]`.
- **Zielordner wird vor jedem Lauf gelöscht.** Der Watchdog löscht `C:\Users\Public\10020115_WinScripts\Win11_C` vollständig und kopiert die Quelle neu. Lokale Änderungen an den Skripten gehen verloren. Zum Debuggen eine Kopie anlegen.
- **Internetprüfung:** Abruf von `https://www.google.com/generate_204`, 5 Versuche, 10 Sekunden Pause, 10 Sekunden Timeout. Ohne Verbindung beendet der Watchdog mit Exit-Code 1, nachdem er das Meldungsskript `StartUp\10020115_Win_S_Messagebox.ps1` ausgeführt hat (zuerst aus der Quelle, sonst aus dem Ziel).
- **Das Meldungsskript fehlt in diesem Repo:** Im Ordner `StartUp` liegt `10020115_Win_S_MessageWLAN.ps1`, der Watchdog sucht aber `10020115_Win_S_Messagebox.ps1`. Liegt die Datei nicht in der Sync-Quelle, endet der Watchdog bei fehlender Verbindung mit `Message-Script nicht gefunden (…)`, und die WLAN-Meldung erscheint nicht. TODO: prüfen, ob die Datei in der Sync-Quelle existiert oder der Name im Watchdog angepasst werden muss.
- **Kodierung:** Vor der Ausführung stellt `Ensure-AllScriptsEncoding` die Kodierung aller Skripte um. Probleme mit Umlauten oder Parsefehlern nach dem Sync kommen oft aus dieser Stelle.
- **Ausführungsreihenfolge:** alle `.ps1` unter `Software\Scripte` rekursiv, außer unter `StartUp`, in der Reihenfolge, in der das Dateisystem sie liefert. Es gibt keine Abhängigkeiten zwischen den Skripten.
- **Rechte:** Der Task läuft mit `-RunLevel Highest` für den eingerichteten Benutzer. Die Richtlinien-Skripte prüfen auf Administrator und brechen sonst ab.

---

## Fehlermeldungen

### Watchdog und Einrichtung

| Meldung | Ursache | Lösung |
|---|---|---|
| `Keine Internetverbindung (Versuch i/5): <detail>` / `Keine Internetverbindung nach 5 Versuchen.` | kein Netz oder `www.google.com` blockiert (Proxy, Firewall) | Netz und Proxy prüfen. Der Lauf endet mit Exit-Code 1 |
| `Abbruch: Keine Internetverbindung. Versuche Message-Script auszuführen.` | Folge der obigen Meldung | siehe oben |
| `Message-Script nicht gefunden (Quelle: …, Ziel: …).` | Meldungsskript fehlt in Quelle und Ziel | Datei bereitstellen oder Pfad `$MessageScriptRelative` anpassen |
| `Ausführen des Message-Scripts (Quelle/Ziel) fehlgeschlagen: <detail>` | Meldungsskript wirft einen Fehler | Skript einzeln starten |
| `Quellpfad existiert nicht: <pfad>` | `$Source` zeigt ins Leere (OneDrive nicht synchronisiert, anderer Benutzername) | Sync-Ordner und `$Source` prüfen, der Lauf endet mit Exit-Code 1 |
| `Loeschen des Zielordners fehlgeschlagen: <detail>` | Dateien im Ziel gesperrt (Skript läuft noch, Virenscanner) | laufende Skripte beenden, erneut starten |
| `Erstellen des Zielordners fehlgeschlagen: <detail>` | keine Rechte unter `C:\Users\Public` | Rechte prüfen |
| `Kopiervorgang fehlgeschlagen: <detail>` | Quelle nicht vollständig lokal verfügbar (OneDrive „Dateien bei Bedarf“), Pfad zu lang | Ordner in OneDrive „Immer auf diesem Gerät behalten“ |
| `Attrib entfernen fehlgeschlagen: …` / `Fehler beim Entfernen von Attributen: …` / `Attribute setzen fehlgeschlagen: …` | einzelne Datei lässt sich nicht umstellen | meist harmlos (Warnung) |
| `Ensure-AllScriptsEncoding: RootPath existiert nicht: <pfad>` / `ScriptRoot existiert nicht: <pfad>` | Kopie war nicht vollständig | Kopierfehler weiter oben im Log suchen |
| `Fehler in Ensure-AllScriptsEncoding: …` / `Ensure-AllScriptsEncoding fehlgeschlagen: <datei> - <detail>` | Umkodierung einer Datei scheiterte | Datei prüfen |
| `Keine Skripte gefunden (nach Ausschluss von StartUp).` | Quelle enthält keine `.ps1` außerhalb von `StartUp` | Quelle prüfen |
| `Fehler in <Skript>: <detail>` | Start des Kindprozesses scheiterte (nicht ein Fehler im Skript selbst) | `powershell.exe` im Pfad? |
| `Kein interaktiver Benutzer erkannt. Bitte PowerShell normal (nicht als Administrator) öffnen.` (`WatchDogInstall.ps1`, Exit 1) | kein angemeldeter Benutzer (zum Beispiel per Remote ohne Sitzung) | im Benutzerkontext starten |
| `Dieses Skript läuft als Administrator. Der Run-Key wird im Admin-Profil gesetzt …` (Warnung) | als Administrator gestartet | ohne „Als Administrator“ ausführen |
| `Watchdog-Skript nicht gefunden: <pfad>` (Warnung) | Standardpfad fehlt noch (erste Einrichtung) | Skripte zuerst kopieren oder `-ScriptPath` setzen |
| `Fehler beim Registrieren der Aufgabe: <detail>` (Exit 2) | keine Berechtigung für die Aufgabenplanung | Rechte prüfen |

### WLAN-Erinnerung (`10020115_Win_S_MessageWLAN.ps1`)

| Symptom | Ursache | Lösung |
|---|---|---|
| Meldebox erscheint nie | Das Skript zählt, an wie vielen Kalendertagen es **gelaufen** ist (einmal pro Tag, Zähldatei `10020115_Win_R_WLANCounter.txt` mit Inhalt `<zähler>;<datum>`) und zeigt die Box ab `$threshold` (20). Es prüft selbst keine WLAN-Verbindung | Es muss vom Watchdog oder einer Aufgabe regelmäßig aufgerufen werden (siehe Abschnitt „Das Meldungsskript fehlt …“). Zähldatei lesen |
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
| `Firewall-Regel nicht gefunden (uebersprungen): <regel>` / `Fehler bei Regel '<regel>': …` / `Public-Profil-Haertung teilweise fehlgeschlagen: …` / `Status-Abfrage fehlgeschlagen: …` | Firewall | Regel existiert in dieser Windows-Version nicht | Warnung, keine Aktion nötig |

### Update, Programme, Software

| Meldung | Skript | Ursache | Lösung |
|---|---|---|---|
| `LOGFILE-Parameter fehlt.` | Update (Hintergrundmodus) | Skript mit `-RUN_IN_BACKGROUND` ohne `-LOGFILE` gestartet | ohne Schalter starten, die beiden Parameter setzt das Skript selbst |
| `Import-Module … Logging.psm1` schlägt fehl | mehrere | Modul am festen Pfad nicht vorhanden (Sync noch nicht gelaufen) | Watchdog erneut laufen lassen, Pfad prüfen |
| `Installation von PSWindowsUpdate fehlgeschlagen: …` | Update | kein Netz, PSGallery nicht erreichbar, keine Rechte | `Install-Module PSWindowsUpdate -Force` als Administrator |
| `Fehler beim Abrufen der Updates: …` / `Fehler bei der Update-Installation: …` | Update | Windows-Update-Dienst gestört | `Get-WindowsUpdateLog`, Dienst `wuauserv` prüfen |
| `Treiberupdate-Einstellung konnte nicht gesetzt werden: …` | Update | Registry-Recht | Warnung |
| `winget nicht gefunden – Software-Updates werden uebersprungen.` | Update | App-Installer fehlt | Microsoft „App Installer“ installieren |
| `Fehler bei Winget-Updates: …` | Update | winget-Fehler | `winget upgrade` von Hand |
| `Keine klassischen Programme gefunden.` / `Fehler beim Auslesen klassischer Programme: …` / `Fehler beim Auslesen der UWP-Apps: …` | Log_Software, Win_A_Install | Registry bzw. AppX-Abfrage lieferte nichts | Warnung bzw. Details lesen |
| `Registry-Pfad nicht gefunden: <pfad>` / `Fehler beim Lesen eines Registry-Eintrags: …` / `Verzeichnis konnte nicht erstellt werden: <pfad> - …` | Win_A_Install | Pfad oder Rechte | Details lesen |
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
| `FEHLER: Geraet mit MAC <mac> nicht gefunden.` (Exit 1) | SSH_toMac | Gerät nicht im lokalen Subnetz oder aus | Gerät einschalten, `-TargetMac` im Format `XX-XX-XX-XX-XX-XX`, `nmap` installieren für gründlicheren Scan |

---

## Checkliste „Skript wirkt nicht“

1. **Läuft der Watchdog?** Aufgabenplanung → `GPO-Watchdog` → letzte Ausführung. Run-Key `GPOWatchdogStarter` vorhanden (`Get-ItemProperty …`)?
2. **Aktuelles Watchdog-Log lesen:** neueste Datei unter `C:\Users\Public\10020115_WinScripts\Logs\`. Steht dort `Keine Internetverbindung`, `Quellpfad existiert nicht` oder `Kopiervorgang fehlgeschlagen`? Dann ist der Sync das Problem.
3. **Skript im Ziel vorhanden?** `C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\...`. Skripte unter `StartUp` führt der Watchdog nie aus.
4. **Log des Kindskripts:** Suche nach `+++ <Skriptname> +++` in den Logs und danach nach `[ERROR]`.
5. **Rechte:** Meldung `Administratorrechte erforderlich.`? Skript von Hand in einer Administrator-PowerShell starten.
6. **Einzeln starten:** `powershell.exe -ExecutionPolicy Bypass -File <skript>` und die Ausgabe lesen.
7. **Modul vorhanden:** `Test-Path C:\Users\Public\10020115_WinScripts\Win11_C\Software\Scripte\Logging.psm1`.
8. **Wirkung prüfen:** Richtlinien setzen Registry-Werte, siehe [Rollback-Tabelle in der Nutzung](nutzung.md#rollback) für die betroffenen Schlüssel. Einige Wirkungen (OEM-Branding, Store) werden erst nach einem Neustart sichtbar.

---

## Typische Probleme zwischen den Teilen

| Symptom | Ursache | Lösung |
|---|---|---|
| Nach dem Verschieben des Repos laufen die Skripte nicht | Pfade `C:\Users\Public\10020115_WinScripts\…` und `Software\Scripte\…` sind in mehreren Skripten fest eingetragen | Struktur der Sync-Quelle beibehalten ([Konfiguration](konfiguration.md)) |
| Sync-Quelle für einen anderen Benutzer | `$Source` im Watchdog enthält den Benutzernamen | Pfad pro Gerät anpassen |
| Skripte im Repo anders sortiert als in der Sync-Quelle | Repo-Struktur (`powershell/`) ≠ erwarteter Pfad `Software\Scripte\…` | Quelle muss die Struktur `Software\Scripte\<Ordner>` haben |
| Watchdog läuft, aber keine Wirkung | Kindskript scheiterte, Watchdog meldet trotzdem `Fertig` | Logs der Kindskripte lesen |
