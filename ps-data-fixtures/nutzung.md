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
