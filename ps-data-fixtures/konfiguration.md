# Konfiguration

Die Struktur wird durch die Namenskonvention bestimmt, nicht durch Konfigurationsdateien.

| Thema | Wo beschrieben |
|---|---|
| Aufbau des Verzeichnisbaums | [Verzeichnisindex](verzeichnisindex.md) |
| Dateinamen (`[Kürzel]_[Präfix]_[Suffix]_[Nummer].[Format]`) | [Namenskonvention](namenskonvention.md) |
| Schülerkürzel | 1. Buchstabe Vorname, 1. und 3. Buchstabe Nachname, z. B. `PKE` |
| Präfixe und Suffixe | Java, Python, Datenbanken, Netzwerktechnik, Zahlensysteme, PowerShell mit je zehn Suffixen, vollständige Liste in der [Namenskonvention](namenskonvention.md) |

Die Whitelist `config/project-routes.txt` von [DocFlowEngine](../DocFlowEngine/konfiguration.md) kennt dieselben Präfixe und Suffixe. Neue Fixtures mit Kombinationen, die dort fehlen, löst DocFlowEngine als falsch benannt auf.
