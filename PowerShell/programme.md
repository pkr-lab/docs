# Programme – Verwaltungs- und Inventur-Skripte

> **Hinweis:** Der Ordnername `Porgamme` ist ein historischer Tippfehler (korrekt: `Programme`).

Skripte in diesem Ordner werden **automatisch vom Watchdog** bei jedem Login ausgeführt.

## Dateien

| Skript | Zweck | Als Admin? |
|--------|-------|-----------|
| `10020115_WIN_Log_Software.ps1` | Software-Inventur ins Log schreiben | Nein |
| `10020115_Win_doc_Word.ps1` | Word-Vorlagen (.dotx) installieren und Registry setzen | Nein |

---

## `10020115_WIN_Log_Software.ps1`

**Zweck:** Erstellt eine vollständige Software-Inventur des Systems und schreibt sie ins Logging-System.

**Erfasst:**
- Klassische Programme (aus Registry `HKLM:\...\Uninstall`)
- UWP/Store-Apps (`Get-AppxPackage`)

**Abhängigkeiten:** `Logging.psm1`

```powershell
.\10020115_WIN_Log_Software.ps1
```

---

## `10020115_Win_doc_Word.ps1`

**Zweck:** Verteilt `.dotx`-Vorlagendateien an den Word-Vorlagenordner des Benutzers und registriert den Pfad in der Registry für alle installierten Office-Versionen.

**Parameter:**

| Parameter | Standard | Beschreibung |
|-----------|---------|--------------|
| `$TemplateSourcePath` | `...\Vorlagen` | Quellordner mit .dotx-Dateien |
| `$OfficeVersions` | `16.0, 15.0, 14.0, 12.0` | Office-Versionen für Registry |
| `$OpenInWord` | `$true` | Vorlagen einmal in Word öffnen (Cache) |
| `$LogLevel` | `INFO` | Log-Detailtiefe |

**Was wird am PC geändert:**
- Kopiert `.dotx`-Dateien nach `%APPDATA%\Microsoft\Templates`
- Setzt `PersonalTemplates` + `USERTEMPLATES` in der Office-Registry
- Startet Word-COM-Instanz kurz zum Einlesen der Vorlagen

```powershell
.\10020115_Win_doc_Word.ps1
.\10020115_Win_doc_Word.ps1 -OpenInWord $false  # Ohne Word-Start
```

**Rollback:**
```powershell
Remove-Item "$env:APPDATA\Microsoft\Templates\*.dotx"
```
