# Tests

`tests/DocFlowEngine.Tests.ps1` enthält [Pester](https://pester.dev/)-Tests für alle Teilmodule.
Stand der Prüfung (2026-11-15): 104 `It`-Blöcke in 30 `Describe`- und 9 `Context`-Blöcken. Mit den
`-ForEach`-Testfällen ergibt das 120 Tests, alle grün mit Pester 5.7.1 und 6.2.0 unter PowerShell 7.4.

## Voraussetzungen

Pester ist **kein** Bestandteil der eigentlichen DocFlowEngine-Laufzeit (siehe
[Voraussetzungen im Haupt-README](installation.md#voraussetzungen) - "keine externen Module nötig"
bezieht sich auf `Invoke-DocFlowEngine` selbst). Für die Tests wird Pester 5 oder neuer benötigt,
installierbar ohne Admin-Rechte:

```powershell
Install-Module -Name Pester -MinimumVersion 5.5.0 -Scope CurrentUser -Force -SkipPublisherCheck
Install-Module -Name PSScriptAnalyzer -Scope CurrentUser -Force
```

`-Scope CurrentUser` installiert nur für den aktuellen Benutzer - kein Admin-/Root-Kontext nötig
(siehe [Compatibility-und-Admin-Rechte.md](compatibility-und-admin-rechte.md)). `-SkipPublisherCheck`
ist nur relevant, falls bereits eine ältere, aus einer anderen Quelle signierte Pester-Version
vorhanden ist (z. B. die mit Windows ausgelieferte Pester 3.4.0).

## Ausführen

```powershell
Invoke-Pester -Path ./tests/DocFlowEngine.Tests.ps1
```

Ausführlichere Ausgabe:

```powershell
$config = New-PesterConfiguration
$config.Run.Path = './tests/DocFlowEngine.Tests.ps1'
$config.Output.Verbosity = 'Detailed'
Invoke-Pester -Configuration $config
```

Statische Analyse mit denselben Regeln wie in der CI:

```powershell
Invoke-ScriptAnalyzer -Path ./scripts -Recurse -Settings ./PSScriptAnalyzerSettings.psd1
```

## CI

`.github/workflows/test.yml` läuft bei Pull Requests gegen `main`, bei Pushes auf `main` und manuell
(`workflow_dispatch`) auf `ubuntu-latest` mit `pwsh`:

| Schritt | Inhalt |
|---|---|
| Installation | Pester 5.7.1 und PSScriptAnalyzer 1.25.0 aus der PowerShell Gallery (`-Scope CurrentUser`, feste Versionen) |
| PSScriptAnalyzer | `Invoke-ScriptAnalyzer -Path ./scripts -Recurse -Settings ./PSScriptAnalyzerSettings.psd1`; jeder Befund lässt den Job scheitern |
| Pester | `Invoke-Pester` auf `./tests` mit `Run.Exit = $true` und Ausgabe `Detailed` |

`PSScriptAnalyzerSettings.psd1` aktiviert die Standardregeln ab Stufe `Warning`. Ausgeschlossen sind:

| Regel | Grund |
|---|---|
| `PSUseSingularNouns`, `PSAvoidOverwritingBuiltInCmdlets` | Würden öffentliche Funktionsnamen ändern (`Get-SourceFiles`, `Write-Log`, …). |
| `PSUseShouldProcessForStateChangingFunctions` | Das Modul nutzt den eigenen `-DryRun`-Mechanismus. |
| `PSAvoidUsingWriteHost` | Die Skripte sind CLI-Werkzeuge und schreiben bewusst auf die Konsole. |
| `PSUseToExportFieldsInManifest` | Die Exportliste steht in `Export-ModuleMember` in `DocFlowEngine.psm1`. |
| `PSUseBOMForUnicodeEncodedFile` | Die Skripte sind UTF-8 ohne BOM gespeichert. Ob sie für Windows PowerShell 5.1 auf UTF-8 mit BOM umgestellt werden, ist noch offen. |

Die CI prüft nur unter PowerShell 7 auf Linux. Windows PowerShell 5.1 wird nicht automatisch getestet.

## Was getestet wird

| Bereich | Beispiele |
|---|---|
| Namenskonvention | Optionale Regex-Gruppe (`versiontag`) wird korrekt durch leeren String ersetzt statt als `{versiontag}` stehen zu bleiben; Kürzel-Extraktion (`Get-FileInitials`). |
| Pro-Datei-Hinweis | Individuelle statt geteilter Hinweisdatei; keine erneute Erzeugung, falls schon vorhanden. |
| Präfix/Suffix-Whitelist | Bekannte Kombination wird geroutet; unbekannte Kombination erzeugt Hinweis statt Kopie **und** wird nicht automatisch in die Registry aufgenommen; ohne konfigurierte Registry gilt das alte, offene Verhalten. |
| "Korrigiert"-Rücklauf | Kürzel-Registry-Rundlauf (Schreiben/Lesen); Rückkopie inkl. Dedup über zwei Läufe; unbekanntes Kürzel → Hinweisdatei statt Kopie; bereits im `Korrigiert`-Ordner liegende Dateien werden ignoriert. |
| Multi-Machine-Fixes | Dedup-Key hängt vom konfigurierten (ggf. wildcardhaltigen) `source.path` ab, nicht vom lokal aufgelösten Rechnerpfad; Ziel-Existenz-Check verhindert blindes Überschreiben; Dateien im `Korrigiert`-Unterordner werden beim Quell-Scan übersprungen. |
| Lock-Mechanismus | Zweiter Lauf wird bei frischer Lock-Datei abgewiesen; abgelaufene Lock-Datei wird übernommen; `Unlock-DocFlowRun` entfernt die Datei zuverlässig. |
| Config-Defaults | `Import-DocFlowConfig` setzt Defaults für alle optionalen Felder. Ausdrücklich gesetzte `$false`/`0`-Werte bleiben erhalten, wo nur fehlende Schlüssel ergänzt werden; leere Suffixe, Meldungen und Pfade werden ersetzt. Die alten Namen `Load-Config`, `Load-State` und `Ensure-TargetDirectories` funktionieren als Aliase, auch über das Manifest. |
| Zustandsdatei | Einelementige und leere `targets`-Listen bleiben nach Speichern und erneutem Laden Listen. |
| Whitelist-Meldungen | Wortlaut der Warnung für unbekanntes Präfix, unbekanntes Suffix und nicht erlaubte Kombination. |
| Doppelabgabe | Versionsvorschlag zählt korrigierte Versionen mit, auch bei eigenem `reviewMarker.pattern`. |
| Sonderzeichen | Dateien und Unterordner mit `[ ]` im Namen werden kopiert (kein Wildcard-Abgleich). |
| Namensvorschläge | `Get-DocFlowNameProposal` für Paare, zusammengeschriebene Namen, nur Suffix, nur Präfix, mehrdeutige Fälle, flache und fehlende Registry, Versions- und Kopie-Bereinigung, bekannte Initialen und Platzhalter; `New-DocFlowSubmissionStem`. |
| `Resolve-PathOrAbsolute` | Existierende Pfade, Wildcard-Auflösung über das nächste vorhandene Elternverzeichnis, Fehler ohne solches Verzeichnis, relative Pfade. |
| `Invoke-DocFlowEngine` | Ende-zu-Ende: DryRun ändert nur das Log; echter Lauf schreibt Kopien, Hinweise, Zustand, Kürzel-Registry und Log und gibt die Sperre frei; zweiter Lauf überspringt; aktive fremde Sperre blockiert den Lauf. |
| Laufkontext | `New-DocFlowRunContext` bündelt Zustand, aufgelöste Pfade, Whitelist, Kürzel-Registry und Ausschlussordner; `Invoke-DocFlowCopyForward` kopiert damit wie `Copy-NewFiles`. Die übrigen `Copy-NewFiles`-Tests laufen weiter über die Kompatibilitätshülle. |
| Skripte | `Rename-ExistingAufgaben.ps1` (Vorschau, `-Apply`, `-Undo`, `-MarkAsProcessed`); `Invoke-OneDriveSync.ps1` ohne `OneDrive.exe` (Warnung, Exit-Code 0). |
| Rename-Hilfsfunktionen | `Get-DocFlowFolderInitials` mit allen drei Quellen und dem mehrdeutigen Fall; `Undo-DocFlowRename` mit fehlender Datei, belegtem altem Namen, nicht ausgeführten Zeilen und fehlendem Bericht; `ConvertTo-DocFlowKuerzelByFolder`. |
| `Get-DocFlowRelativePath` | Plattformunabhängige, reine String-basierte Berechnung (kein `[Uri]`-basierter Ansatz, der unter Linux/macOS ohne `file://`-Schema fehlschlägt). |

## Testaufbau

- Jeder Test arbeitet in Pesters `$TestDrive` (temporäres, automatisch aufgeräumtes Verzeichnis) -
  es werden keine Dateien im Projektverzeichnis selbst angelegt.
- Datei-/Ordnernamen enthalten durchgehend `New-Guid`, um Kollisionen zwischen Tests im selben
  `$TestDrive`-Lauf zu vermeiden (`$TestDrive` wird nicht zwischen einzelnen `It`-Blöcken geleert).
- Die Kernfunktionen werden direkt gegen das echte Dateisystem aufgerufen. Gemockt wird nur
  `Copy-Item` (Kopierwiederholung bei gesperrten Dateien) und `Write-Log` (Prüfung des Wortlauts
  von Warnungen).

## Manuelle End-to-End-Prüfung

Der Ende-zu-Ende-Ablauf ist inzwischen auch automatisiert (`Describe 'Invoke-DocFlowEngine (Ende-zu-Ende)'`).
Zusätzlich wurde der komplette Ablauf (`Invoke-DocFlowEngine` gegen
eine Test-Konfiguration mit echten Fixture-Ordnern) mehrfach manuell durchgespielt: Vorwärtslauf,
wiederholter Lauf (Idempotenz), Korrigiert-Rücklauf, unbekanntes Kürzel, aktiver Lock-Konflikt,
bekannte/unbekannte Präfix-Suffix-Kombinationen. Diese Fixtures sind bewusst nicht Teil des Repos
(reine Wegwerf-Verzeichnisse während der Entwicklung) - die dauerhafte Absicherung liegt in den
Pester-Tests oben.
