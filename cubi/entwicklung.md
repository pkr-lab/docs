# Entwicklung

Aufbau, Tests und Erweiterung von cubi. Für den Betrieb siehe [Debugging und Fehlersuche](debugging.md).

Code: [pkr-lab/cubi](https://github.com/pkr-lab/cubi)

---

## Voraussetzungen und Start aus dem Checkout

```bash
git clone https://github.com/pkr-lab/cubi.git && cd cubi
git config core.hooksPath .githooks      # Commit-Hook: Conventional Commits
./bin/cubi --help
```

`bin/cubi` ergänzt `src/` im Suchpfad und ruft `cubi.main.main()` auf. Strg+C beendet mit Exit-Code 130. Zur Laufzeit gibt es nur `rich` und `pyyaml` als Abhängigkeiten, es gibt keine `requirements.txt` und kein Build.

---

## Tests

Die Tests nutzen `unittest` (kein `pytest`).

```bash
python3 -m unittest discover -s tests      # alle, 201 Tests
```

| Datei | Prüft |
|---|---|
| `test_argocd_apps.py` | Normalisieren der Applications, ApplicationSet-Regeln, Zustandsspeicher |
| `test_autosync.py`, `test_feature_testing.py`, `test_scale.py` | Planlogik, Patches, Nachkontrolle, Ausgabe der Zusammenfassung (mit gemocktem `kubectl`) |
| `test_choice.py`, `test_term.py` | Auswahl, Viewport, Tastaturdekodierung |
| `test_finish.py` | Programmende nach Änderung (`Finished`) |
| `test_inline_pty.py`, `test_no_fullscreen.py` | Verhalten in einem Pseudo-Terminal: kein Alternate Screen, kein Löschen, Bereich wird ersetzt |
| `helpers.py` | Hilfsfunktion `run_in_pty`, Pfad auf `src/` |

Die Tests rufen weder Cluster noch GitHub auf: Unterprozesse sind mit `unittest.mock` ersetzt. Für die Module `connect`, `tailscale`, `github`, `kube`, `helm`, `tokens` und `auth` gibt es keine eigenen Tests (Stand der Prüfung).

---

## Einen neuen Bereich ergänzen

1. Modul `src/cubi/modules/<name>.py` mit `run()` anlegen. Tabellen über `ui.table_select`, Menüs über `ui.select` oder `ui.action_menu`, Befehle über `shell.capture`, `ui.show_command` oder `ui.run_command`.
2. Fehler als `CubiError("…")` auslösen, damit `ui.guard` sie als Meldung zeigt. `Finished` beendet cubi nach einer Änderung.
3. In `main.py` in `MODULES` (Name, `run`, Titel, Hinweis) und bei Bedarf in `ALIASES` eintragen.
4. Neue Konfigurationsschlüssel über `config.section("<name>")` lesen und in [Konfiguration](konfiguration.md) dokumentieren.
5. Seite hier ergänzen, mit Abschnitt „Debugging und Fehlersuche“ und Meldungstabelle.

Regeln, die der Code durchgehend einhält: kein Vollbild und kein externes Programm (Pager, Editor, Browser), nur Konfiguration über `targets.yaml`, keine Secrets speichern, Fehlertexte auf Deutsch.

---

## Release

Releases entstehen aus Commits (Conventional Commits). Ablauf: [Releases](releases.md). Der Release-Build (`.github/release/build.sh`) packt einen Tarball als Release-Asset.
