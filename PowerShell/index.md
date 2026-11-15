# PowerShell – Skriptsammlung für Windows-Administration (DLRG-Jugend Andernach)

Automatisierte Windows-Verwaltung: Updates, Richtlinien, Personalisierung, Inventur und Monitoring.  
Ziel ist eine **klare Dokumentation**, **nachvollziehbare Systemänderungen** und **sichere** Ausführung.

> ⚠️ **Wichtige Hinweise**
> - Skripte **zuerst in einer Testumgebung** testen.
> - Skripte mit Systemzugriff (HKLM, Firewall, Policies) **als Administrator** ausführen.
> - Execution Policy: `powershell.exe -ExecutionPolicy Bypass -File .\Skript.ps1`

Repository: [pkr-lab/PowerShell](https://github.com/pkr-lab/PowerShell)

## Überblick

Die Sammlung besteht aus Skripten für Windows-10/11-Clients, die per Watchdog bei jeder Anmeldung aus einer Sync-Quelle (OneDrive/SharePoint) in ein lokales Verzeichnis kopiert und ausgeführt werden. Alle Skripte nutzen das gemeinsame Modul `Logging.psm1`, die meisten zusätzlich `Common.psm1` (Admin-Prüfung, Inventur, Paket-Bootstrap, Konfiguration). Werte je Einrichtung stehen in der lokalen, nicht versionierten `Konfiguration.psd1` ([Konfiguration](konfiguration.md#lokale-konfiguration-konfigurationpsd1)).

| Ordner | Aufgabe | Seite |
|---|---|---|
| `Ausfuehrung/` | einmalige Einrichtungs- und Hilfsskripte (Watchdog installieren, Dateien in eine VM kopieren), vom Watchdog ausgeschlossen | [ausfuehrung.md](ausfuehrung.md) |
| `StartUp/` | Login-Skripte: Erst-Einrichtung, WLAN-Erinnerung, Watchdog | [startup.md](startup.md) |
| `Richtlinien/` | Windows-Policy-Konfiguration (Firewall, PIN, Personalisierung, Store, Branding) | [richtlinien.md](richtlinien.md) |
| `Programme/` | Verwaltungsskripte (Software-Inventur, Word-Vorlagen) | [programme.md](programme.md) |
| `Update/` | Windows-, Treiber- und Winget-Updates | [update.md](update.md) |
| `InstallSoftware/` | Hersteller-Tools für BIOS-Updates (HP, Lenovo) | [installsoftware.md](installsoftware.md) |
| Wurzel | `Logging.psm1`, `Common.psm1`, `Konfiguration.example.psd1`, `SSH_toMac.ps1` (vom Watchdog nicht gestartet) | [konfiguration.md](konfiguration.md) |

## Namensschema der Skripte

`10020115_Win_<Typ>_<Name>.ps1`. `10020115` ist das feste Präfix aller Skripte der Sammlung. Der Typ ordnet das Skript ein: `A` für Einrichtungs- und Installationsskripte, `R` für Richtlinien, `S` für Skripte im Ordner `StartUp`, `doc` und `Log` für Verwaltungsskripte. Der Watchdog führt alle `.ps1` aus den Ordnern `Richtlinien`, `Programme`, `Update` und `InstallSoftware` aus, `StartUp`, `Ausfuehrung` und der Wurzelordner sind ausgenommen.

## Ordnerstruktur

```
powershell/
├── Logging.psm1                  ← Gemeinsames Logging-Modul (von allen Skripten genutzt)
├── Common.psm1                   ← Gemeinsame Funktionen (Admin-Prüfung, Inventur, Konfiguration)
├── Konfiguration.example.psd1    ← Vorlage für die lokale Konfiguration.psd1 (nicht im Repo)
├── SSH_toMac.ps1                 ← SSH-Verbindung per MAC-Adresse aufbauen
│
├── Ausfuehrung/                  ← Einmalige Einrichtungs- und Hilfsskripte (Watchdog-Ausführung AUSGESCHLOSSEN)
│   ├── DateienSendenVM.ps1       ← Dateien in Hyper-V-VM kopieren, GPO Export/Import
│   └── WatchDogInstall.ps1       ← Watchdog als geplante Aufgabe einrichten
│
├── InstallSoftware/              ← Hersteller-spezifische Tool-Installation
│   └── 10020115_Win_A_InstallBIOStools.ps1  ← HP (HPCMSL) oder Lenovo System Update
│
├── Programme/                    ← Verwaltungs-Skripte (via Watchdog ausgeführt)
│   ├── 10020115_Win_Log_Software.ps1        ← Software-Inventur
│   └── 10020115_Win_doc_Word.ps1            ← Word-Vorlagen (.dotx) verteilen
│
├── Richtlinien/                  ← Windows-Policy-Konfiguration (via Watchdog ausgeführt)
│   ├── 10020115_Win_R_Firewall.ps1          ← Windows-Firewall konfigurieren
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

## Architektur & Automatisierung

```
Bei jedem Benutzer-Login:
  Watchdog (10020115_Win_S_Watchdog.ps1)
    │
    ├── Liest WatchdogQuelle aus Konfiguration.psd1
    ├── Prüft Internetverbindung (5 Versuche x 10 Sek.)
    ├── Kein Internet → WLAN-Meldebox anzeigen, abbrechen
    │
    ├── Kopiert Skripte von Sync-Quelle nach
    │   C:\Users\Public\10020115_WinScripts\Win11_C\
    │
    └── Führt alle .ps1 in Unterordnern von Software\Scripte\ aus (außer StartUp\, Ausfuehrung\)
        und wertet ihren Exit-Code aus:
          ├── Richtlinien\*     (Policies, Firewall, OEM-Branding)
          ├── Programme\*       (Inventur, Word-Vorlagen)
          ├── Update\*          (Windows + Winget Updates)
          └── InstallSoftware\* (BIOS-Tools je nach Hersteller)
```

**Einrichten des Watchdogs** (einmalig, als normaler Benutzer):
```powershell
.\Ausfuehrung\WatchDogInstall.ps1
```

## Sicherheit & Verantwortungsbereich

- Skripte **vor Produktionseinsatz** in Testumgebung prüfen.
- Alle Systemänderungen (Registry, Dienste, Firewall) dokumentieren.
- Wiederherstellungsoptionen bereitstellen (Systemwiederherstellungspunkt, GPO-Backup).
- Gestaffelte Rollouts: Pilot → breiter Rollout.

## Weitere Skripte

| Bereich | Inhalt | Seite |
|---|---|---|
| `bash/git-pull-push/` | Bash-Skript: Git-Repository aktualisieren, bei neuer Version nach `ZIEL` kopieren und einen Befehl ausführen | [bash-git-pull-push.md](bash-git-pull-push.md) |

Im Repo liegen die PowerShell-Skripte unter `powershell/`, das Bash-Skript unter `bash/`. Die Pfade in dieser Doku sind relativ zu `powershell/` bzw. zu `Software\Scripte\` auf dem Gerät.

## Qualitätsprüfung

Die CI (`.github/workflows/lint.yml`) prüft jeden Pull Request:

| Prüfung | Version | Umfang |
|---|---|---|
| PSScriptAnalyzer | 1.23.0 | alle `.ps1`, `.psm1`, `.psd1` mit `PSScriptAnalyzerSettings.psd1`, muss ohne Befund durchlaufen |
| shellcheck | 0.10.0 (SHA-256 geprüft) | `bash/git-pull-push/git-pull-push.sh`, `.github/release/build.sh`, `.githooks/commit-msg` |

Ausgeschlossene Analyzer-Regeln und ihr Grund:

| Regel | Grund |
|---|---|
| `PSAvoidUsingWriteHost` | Logger und interaktive Werkzeuge geben bewusst farbig auf der Konsole aus |
| `PSUseShouldProcessForStateChangingFunctions` | Die Skripte laufen unbeaufsichtigt, `-WhatIf` und `-Confirm` bringen dort nichts |
| `PSAvoidOverwritingBuiltInCmdlets` | meldet `Write-Log` aus PowerShell 6.1 für Windows, die Skripte laufen unter Windows PowerShell 5.1 |
| `PSProvideCommentHelp` | Die Doku liegt hier, der Code bleibt ohne Kommentare |

Kodierung: `.ps1`, `.psm1` und `.psd1` sind UTF-8 mit BOM (`.editorconfig`), damit Windows PowerShell 5.1 Umlaute richtig liest. Zeilenenden sind LF (`.gitattributes`).

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Debugging und Fehlersuche](debugging.md): Logs, Meldungen je Skript, Verhalten des Watchdogs
- [git-pull-push](bash-git-pull-push.md)
- [Releases](releases.md)


## Fehlersuche

Logs, Meldungen je Skript und Verhalten des Watchdogs: [Debugging und Fehlersuche](debugging.md). Je Ordner steht ein Abschnitt „Fehlersuche“ auf den Seiten [Ausführung](ausfuehrung.md#fehlersuche), [StartUp](startup.md#fehlersuche), [Richtlinien](richtlinien.md#fehlersuche), [Programme](programme.md#fehlersuche), [Update](update.md#fehlersuche) und [InstallSoftware](installsoftware.md#fehlersuche).
