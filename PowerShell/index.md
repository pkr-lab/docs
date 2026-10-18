# PowerShell – Skriptsammlung für Windows-Administration (DLRG-Jugend Andernach)

Automatisierte Windows-Verwaltung: Updates, Richtlinien, Personalisierung, Inventur und Monitoring.  
Ziel ist eine **klare Dokumentation**, **nachvollziehbare Systemänderungen** und **sichere** Ausführung.

> ⚠️ **Wichtige Hinweise**
> - Skripte **zuerst in einer Testumgebung** testen.
> - Skripte mit Systemzugriff (HKLM, Firewall, Policies) **als Administrator** ausführen.
> - Execution Policy: `powershell.exe -ExecutionPolicy Bypass -File .\Skript.ps1`

---

## Ordnerstruktur

```
PowerShell/
├── Logging.psm1                  ← Gemeinsames Logging-Modul (von allen Skripten genutzt)
├── SSH_toMac.ps1                 ← SSH-Verbindung per MAC-Adresse aufbauen
│
├── Ausführung/                   ← Einmalige Einrichtungs- und Hilfsskripte
│   ├── DateienSendenVM.ps1       ← Dateien in Hyper-V-VM kopieren, GPO Export/Import
│   └── WatchDogInstall.ps1       ← Watchdog als geplante Aufgabe einrichten
│
├── InstallSoftware/              ← Hersteller-spezifische Tool-Installation
│   └── 10020115_Win_A_InstallBIOStools.ps1  ← HP (HPCMSL) oder Lenovo System Update
│
├── Porgamme/                     ← Verwaltungs-Skripte (via Watchdog ausgeführt)
│   ├── 10020115_WIN_Log_Software.ps1        ← Software-Inventur
│   └── 10020115_Win_doc_Word.ps1            ← Word-Vorlagen (.dotx) verteilen
│
├── Richtlinien/                  ← Windows-Policy-Konfiguration (via Watchdog ausgeführt)
│   ├── 10020115_Win_R_Firewall.ps1          ← Windows-Firewall konfigurieren (NEU)
│   ├── 10020115_Win_R_Passwort.ps1          ← PIN-Komplexität (Windows Hello)
│   ├── 10020115_Win_R_Personalisierung.ps1  ← Desktop- und Sperrbildschirm-Hintergrund
│   ├── 10020115_Win_R_SoftwareInstallUninstall.ps1  ← Software-Richtlinien zurücksetzen
│   ├── 10020115_Win_R_SystemHilfe.ps1       ← OEM-Branding und Legal Notice
│   └── 10020115_Win_R_WinInstallStore.ps1   ← Microsoft Store deaktivieren
│
├── StartUp/                      ← Login-Skripte (Watchdog-Ausführung AUSGESCHLOSSEN)
│   ├── 10020115_Win_A_Install.ps1           ← Erst-Einrichtung (einmalig)
│   ├── 10020115_Win_S_MessageWLAN.ps1       ← WLAN-Erinnerung nach X Tagen
│   └── 10020115_Win_S_Watchdog.ps1          ← Hauptprozess: Sync + Skript-Ausführung
│
└── Update/                       ← Update-Automatisierung
    └── 10020115_Win_UpdateScript.ps1        ← Windows + Treiber + Winget (Hintergrund)
```

---

## Voraussetzungen

| Anforderung | Details |
|------------|---------|
| Windows | Windows 10 oder Windows 11 (Client) |
| PowerShell | 5.1 oder 7.x |
| Rechte | Lokale Administratorrechte für Systemeingriffe |
| Internet | Für Updates und Modul-Installation (PSWindowsUpdate, HPCMSL) |
| OneDrive/SharePoint | Sync-Quelle für den Watchdog (im Skript konfigurierbar) |

---

## Architektur & Automatisierung

```
Bei jedem Benutzer-Login:
  Watchdog (10020115_Win_S_Watchdog.ps1)
    │
    ├── Prüft Internetverbindung (5 Versuche x 10 Sek.)
    ├── Kein Internet → WLAN-Meldebox anzeigen, abbrechen
    │
    ├── Kopiert Skripte von Sync-Quelle nach
    │   C:\Users\Public\10020115_WinScripts\Win11_C\
    │
    └── Führt alle .ps1 in Software\Scripte\ aus (außer StartUp\):
          ├── Richtlinien\*     (Policies, Firewall, OEM-Branding)
          ├── Porgamme\*        (Inventur, Word-Vorlagen)
          ├── Update\*          (Windows + Winget Updates)
          └── InstallSoftware\* (BIOS-Tools je nach Hersteller)
```

**Einrichten des Watchdogs** (einmalig, als normaler Benutzer):
```powershell
.\Ausführung\WatchDogInstall.ps1
```

---

## Gemeinsames Logging-Modul (`Logging.psm1`)

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

---

## Sicherheit & Verantwortungsbereich

- Skripte **vor Produktionseinsatz** in Testumgebung prüfen.
- Alle Systemänderungen (Registry, Dienste, Firewall) dokumentieren.
- Wiederherstellungsoptionen bereitstellen (Systemwiederherstellungspunkt, GPO-Backup).
- Gestaffelte Rollouts: Pilot → breiter Rollout.

---

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| Skript wird blockiert | `-ExecutionPolicy Bypass` verwenden |
| PSWindowsUpdate nicht gefunden | `Install-Module PSWindowsUpdate -Force` als Admin |
| Watchdog startet nicht | `WatchDogInstall.ps1` als normaler Benutzer (nicht als Admin) ausführen |
| Logs leer / nicht vorhanden | Pfad `C:\Users\Public\10020115_WinScripts\Logs\` manuell anlegen |
| OEM-Branding nicht sichtbar | Windows neu starten |
| WLAN-Meldebox erscheint nicht | Zähldatei prüfen: `C:\Users\Public\10020115_WinScripts\10020115_Win_R_WLANCounter.txt` |

---

## Rollback

| Aktion | Befehl |
|--------|--------|
| Watchdog-Task entfernen | `Unregister-ScheduledTask -TaskName "GPO-Watchdog" -Confirm:$false` |
| Run-Key entfernen | `Remove-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name GPOWatchdogStarter` |
| Store-Richtlinie entfernen | `Remove-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" -Name RemoveWindowsStore` |
| PIN-Richtlinie entfernen | `Remove-Item "HKLM:\SOFTWARE\Policies\Microsoft\PassportForWork\PINComplexity" -Recurse -Force` |
| Software-Richtlinien | `.\Richtlinien\10020115_Win_R_SoftwareInstallUninstall.ps1` |
| Legal Notice entfernen | Registry: `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` → `LegalNoticeCaption` + `LegalNoticeText` löschen |
| OEM-Branding entfernen | `Remove-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\OEMInformation" -Recurse` |

---

## Lizenz

MIT – Nutzung auf eigene Verantwortung.

## Mitwirken

Pull Requests und Issues sind willkommen.  
Bitte alle Systemänderungen im zugehörigen Unterordner-README dokumentieren.
