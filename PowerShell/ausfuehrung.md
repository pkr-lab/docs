# Ausführung – Hilfsskripte für Einrichtung & Deployment

Dieser Ordner enthält Skripte, die **einmalig oder manuell** ausgeführt werden – nicht durch den Watchdog-Prozess.

## Dateien

| Skript | Zweck | Als Admin? |
|--------|-------|-----------|
| `DateienSendenVM.ps1` | Dateien in Hyper-V-VM kopieren, GPO Export/Import | Ja |
| `WatchDogInstall.ps1` | Watchdog als geplante Aufgabe registrieren | Nein (Benutzerkontext!) |

---

## `DateienSendenVM.ps1`

**Zweck:** Interaktives Hilfsskript für die Arbeit mit der Hyper-V-Testmaschine.

**Funktionen:**
- Datei vom Host in die VM kopieren (Hyper-V Guest Services)
- Hyper-V Local Account Dialog öffnen
- GPO exportieren (Backup)
- GPO importieren (Restore)

**Voraussetzungen:**
- Hyper-V-Feature aktiviert
- RSAT: GroupPolicy-Modul (für GPO-Befehle)
- Administratorrechte

```powershell
# Interaktiv ausführen
.\DateienSendenVM.ps1

# Mit eigenen Parametern
.\DateienSendenVM.ps1 -VMName "MeineVM" -GpoName "MeineGPO" -GpoBackupPath "D:\GPO-Backups"
```

---

## `WatchDogInstall.ps1`

**Zweck:** Registriert den Watchdog (`10020115_Win_S_Watchdog.ps1`) als geplante Aufgabe, die bei jedem Benutzer-Login automatisch startet.

> ⚠️ **Wichtig:** Muss im **interaktiven Benutzerkontext** ausgeführt werden – NICHT als Administrator. Der Run-Key und der Task werden im Benutzerprofil angelegt.

**Was wird geändert:**
- Neuer Scheduled Task `GPO-Watchdog` (ohne Trigger, per Run-Key gestartet)
- Registry-Eintrag: `HKCU\...\Run\GPOWatchdogStarter`

```powershell
# Standard-Ausführung (normales PowerShell-Fenster, nicht "Als Administrator")
.\WatchDogInstall.ps1

# Mit abweichendem Skriptpfad
.\WatchDogInstall.ps1 -ScriptPath "C:\MeinPfad\Watchdog.ps1"
```

**Rollback:**
```powershell
Unregister-ScheduledTask -TaskName "GPO-Watchdog" -Confirm:$false
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "GPOWatchdogStarter"
```

---

## Fehlersuche

Die Meldungen aller Skripte sammelt [Debugging und Fehlersuche](debugging.md), die Logdateien liegen unter `C:\Users\Public\10020115_WinScripts\Logs\`.

| Skript | Symptom | Ursache und Lösung |
|---|---|---|
| `WatchDogInstall.ps1` | `Kein interaktiver Benutzer erkannt …` (Exit 1) | kein angemeldeter Benutzer (zum Beispiel per Remote-Sitzung ohne Anmeldung), im Benutzerkontext starten |
| | Warnung `Dieses Skript läuft als Administrator …` | als normaler Benutzer starten, sonst landen Run-Key und Task im Admin-Profil |
| | Warnung `Watchdog-Skript nicht gefunden: <pfad>` | Standardpfad existiert noch nicht, Skripte zuerst kopieren oder `-ScriptPath` setzen |
| | `Fehler beim Registrieren der Aufgabe: …` (Exit 2) | fehlende Rechte für die Aufgabenplanung |
| | Watchdog läuft nach dem Login nicht | `Get-ScheduledTask -TaskName GPO-Watchdog`, `Get-ItemProperty HKCU:\Software\Microsoft\Windows\CurrentVersion\Run -Name GPOWatchdogStarter` |
| `DateienSendenVM.ps1` | `Quelldatei nicht gefunden: <pfad>` | `-SourceZip` setzen |
| | `Copy-VMFile` scheitert | Hyper-V-Gastdienste in der VM aktivieren, VM läuft? |
| | `Backup-GPO`/`Import-GPO` nicht gefunden | RSAT: Modul GroupPolicy installieren, als Administrator ausführen |
| | `Ungültige Eingabe.` | gültige Menünummer wählen |
