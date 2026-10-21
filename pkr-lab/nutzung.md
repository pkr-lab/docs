# Nutzung

Änderungen an der Profilseite macht man in der `README.md` per Pull Request. Nach dem Merge erscheint die neue Fassung sofort auf dem GitHub-Profil. Neue Projekte werden zusätzlich in der [Projektübersicht](index.md#überblick) dieser Doku ergänzt.

## Debugging und Fehlersuche

Die Profilseite besteht nur aus einer `README.md`. Stand der Prüfung (2026-10-21): Das Repo liegt nicht unter `~/gitlab`, die Angaben stützen sich auf die vorhandene Doku (TODO: prüfen gegen das Repo selbst).

| Symptom | Ursache | Lösung |
|---|---|---|
| Profilseite zeigt die README nicht | Repo `pkr-lab/pkr-lab` nicht öffentlich oder README heißt anders | Sichtbarkeit und Dateiname `README.md` prüfen ([Installation](installation.md)) |
| Abzeichen erscheinen nicht | Shields.io nicht erreichbar oder URL mit falschem Namen (`pkr--lab` mit doppeltem Bindestrich) | Bild-URL im Browser öffnen, [Konfiguration](konfiguration.md) |
| Änderung nicht sichtbar | Cache von GitHub oder noch nicht gemergt | nach dem Merge neu laden |
| Link auf Projekt-Doku führt ins Leere | Doku-Pfad geändert | Links gegen die [Projektübersicht](../README.md#projekte) abgleichen |
