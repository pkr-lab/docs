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
