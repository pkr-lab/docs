# Nutzung

## Als Quelle für DocFlowEngine

Ein Testlauf mit DocFlowEngine, ohne echte Daten zu berühren:

1. Das Repo in ein Arbeitsverzeichnis klonen (Kopie, nicht das OneDrive-Original).
2. In `config/docflow-config.psd1` von DocFlowEngine `sources[].path` auf einen Austauschordner zeigen lassen, z. B. `ps-data-fixtures/IT-Ausbildung/FI25/Austauschordner`, und `aufgabenRoot` auf ein leeres Testverzeichnis setzen.
3. Zuerst im DryRun laufen lassen:

```powershell
pwsh -NoProfile -ExecutionPolicy Bypass -File ./scripts/DocFlowEngine.ps1 -DryRun
```

Details: [DocFlowEngine Nutzung](../DocFlowEngine/nutzung.md).

## Fixtures ergänzen

- Neue Dateien nach der [Namenskonvention](namenskonvention.md) benennen und in den Ordner des passenden Lernenden legen.
- Erfundene Namen verwenden, keine echten Personen.
- Neue Ordner im [Verzeichnisindex](verzeichnisindex.md) eintragen.

## Debugging und Fehlersuche

Das Repo enthält nur Daten und keinen Code. Fehlerbilder entstehen beim Zusammenspiel mit [DocFlowEngine](../DocFlowEngine/index.md), dessen Meldungen auf der Seite [DocFlowEngine: Debugging](../DocFlowEngine/debugging.md) stehen. Stand der Prüfung (2026-10-21): Das Repo liegt nicht unter `~/gitlab`, die folgenden Angaben stützen sich auf die vorhandene Doku und den Code von DocFlowEngine (TODO: prüfen gegen das Repo selbst).

| Symptom | Ursache | Lösung |
|---|---|---|
| DocFlowEngine meldet für eine Fixture `passt nicht zum Schema …` | Dateiname verletzt die [Namenskonvention](namenskonvention.md) (Leerzeichen, Bindestrich, Umlaut, fehlender Teil) | Datei umbenennen. Schema: `initialen[_v<version>]_praefix_suffix_aufgabennummer` |
| `Präfix '…' ist nicht in der Whitelist … eingetragen` oder `Suffix '…' ist für Präfix '…' nicht …` | neue Kombination, die `config/project-routes.txt` von DocFlowEngine nicht kennt | Whitelist ergänzen ([DocFlowEngine: Konfiguration](../DocFlowEngine/konfiguration.md)) |
| Fixtures werden nicht kopiert | Datei jünger als `minFileAgeMinutes` (Standard 2), bereits im Zustand `.docflow-state.json` | Zustandsdatei der Testumgebung löschen, kurz warten |
| Lauf ändert echte Daten | `sources[].path` zeigt auf das OneDrive-Original | immer auf eine Kopie zeigen, zuerst `-DryRun` ([Nutzung](#als-quelle-für-docflowengine)) |
| macOS-`.DS_Store` stört | versioniert, ohne Bedeutung | ignorieren |
| Unterschiedliche Ergebnisse auf zwei Rechnern | gemeinsamer Zustand fehlt | [Multi-Machine-Setup](../DocFlowEngine/konzepte/multi-machine-setup.md) |
