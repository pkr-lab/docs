# ps-data-fixtures

Testdaten für PowerShell-Skripte: eine künstliche Ordner- und Dateistruktur im Stil eines SharePoint-Bereichs für die IT-Ausbildung mit drei Jahrgängen (FI23, FI24, FI25), fünf Fachbereichen und rund 750 Beispieldateien (PDF und DOCX). Sie dient dazu, Skripte wie [DocFlowEngine](../DocFlowEngine/index.md) ohne echte Daten auszuprobieren.

Repository: [pkr-lab/ps-data-fixtures](https://github.com/pkr-lab/ps-data-fixtures)

## Überblick

```
IT-Ausbildung/
├── FI23/, FI24/, FI25/            # Jahrgänge, je mit Austauschordner/<Fachbereich>/<Vorname_Nachname_HEXCODE>/
├── Gemeinschaftliche_Ressourcen/
├── Lehrplan/                      # Lehrpläne und Kompetenzmatrix
├── Ressourcen/, Vorlagen/, Verwaltung/
└── Tooling/                       # Skripts/, Templates/, Checklists/ (leer, mit .gitkeep)
```

| Fachbereich | Bedeutung |
|---|---|
| AWIS | Anwendungsentwicklung mit Webtechnologien, iOS, Server |
| FIAE | Fachinformatiker/in Anwendungsentwicklung |
| FISI | Fachinformatiker/in Systemintegration |
| FIDP | Fachinformatiker/in Digitale Prozesse |
| MASE | Mathematik und Softwareentwicklung |

Die Namen der Lernenden sind überwiegend erfunden. Jeder hat einen eigenen Ordner `Vorname_Nachname_HEXCODE` mit Dateien nach der [Namenskonvention](namenskonvention.md), z. B. `PKE_Java_ArrayList_001.pdf`.

Die Ordner `FI23`, `FI24`, `FI25`, `Gemeinschaftliche_Ressourcen`, `Lehrplan` und `Tooling` enthalten eine eigene `README.md`; sie gehören zu den Testdaten und bleiben im Repo.

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Verzeichnisindex](verzeichnisindex.md)
- [Namenskonvention](namenskonvention.md)
- [Releases](releases.md)
