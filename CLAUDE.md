# CLAUDE.md — Arbeitsregeln für pkr-lab-docs

Zentrales Dokumentations-Repo aller pkr-lab-Projekte. Sprache ist **Deutsch**. Die Code-Repos enthalten keine Doku außer einer kurzen `README.md`; alles andere gehört hierher.

## Aufbau

- Ein Ordner pro Projekt, benannt wie das Repo. Die capulus-Projekte liegen gebündelt unter `capulus/`: `capulus/capulus-core/`, `capulus/capulus-ansible/`, `capulus/capulus-ios/`, `capulus/capulus-operator/`. Die übrigen liegen direkt im Wurzelverzeichnis: `cubi/`, `ui-terminal-config/`, `DocFlowEngine/`, `PowerShell/`, `ps-data-fixtures/`, `EDV-Kretzer/`, `github-workflows/`, `pkr-lab/`.
- Die Doku von capulus-wiki-sync liegt in `capulus/capulus-core/wiki-sync.md` (Betrieb im Cluster), die Code-Repos verlinken ihre Doku mit den Pfaden `capulus/<repo>/…`.
- Mindestens je Projekt: `index.md` (Zweck, Überblick, Architektur), `installation.md`, `konfiguration.md`, `nutzung.md`, `releases.md`. Größere Projekte bekommen Unterseiten pro Komponente, Rolle oder Modul.
- Docs zu Host-Provisionierung liegen unter `capulus/capulus-ansible/`, cluster- und GitOps-bezogene unter `capulus/capulus-core/`. Incidents bleiben gesammelt unter `capulus/capulus-core/5-incidents/` und `capulus/capulus-core/7-security-incidents/`.

## Doku-Konventionen für capulus-core (`capulus/capulus-core/TEMPLATE.md`)

- Dateiname `<id>-<slug>.md`, `<id>` = Kategorie-Zeichen + 4 Hex-Ziffern, neue IDs in Schritten von `0x10`. Kategorien: `a` OS, `b` Kubernetes/GitOps, `c` Netzwerk/DNS, `d` Sicherheit, `e` extern, `f` CI/CD, `1` Benachrichtigungen, `2` Betrieb/Hardware, `3` Apps, `4` Planung, `5` Incidents, `6` Hintergründe.
- Doc-Typen: A Setup/Runbook, B Konzept/Referenz, C Incident — Skelette in `capulus/capulus-core/TEMPLATE.md`.
- Nur `##`/`###`, `---` zwischen Abschnitten, Tabellen für Konfiguration und Troubleshooting.
- Neue Docs in den Index in `capulus/capulus-core/index.md` (Abschnitt „Dokumentation“) eintragen.
- Docs beschreiben den Ist-Zustand, außer `4-planung/`.
- Begründungen, Fallstricke und Vorfall-Regeln zum Code stehen in `capulus/capulus-core/6-hintergruende/` (Zuordnung Pfad → Doc: `60000-uebersicht.md`). Die Code-Repos sind kommentarfrei.

## Links

- Innerhalb dieses Repos relativ (`../c-netzwerk-dns/c0040-domain-tiers.md`, projektübergreifend `../../capulus-ansible/…`).
- In Code-Dateien eines anderen Repos als absolute GitHub-URL (`https://github.com/pkr-lab/capulus-core/blob/main/…`).
- Bilder liegen beim Projekt (`cubi/pics/`, `capulus/capulus-core/assets/`).
- Prüfen: `python3 scripts/check-doc-links.py`. Die CI führt zusätzlich lychee aus (`lychee.toml`).

## Secrets und interne Daten

Keine Klartext-Secrets, Tokens oder Passwörter in Docs, auch nicht in Beispielen. Platzhalter wie `<TOKEN>` verwenden. gitleaks läuft in der CI; bekannte Platzhalter-Treffer stehen in `.gitleaksignore`.

## Commits und PRs

- **Nie direkt auf `main` arbeiten.** Änderungen gehen nur per PR hinein. Steht das Arbeitsverzeichnis auf `main`, vor der ersten Änderung einen neuen Branch anlegen (`git switch -c docs/<kurzbeschreibung>`, z. B. `docs/capulus-operator-backup`) und nur dort arbeiten. Den Branch legt Claude an, Commit, Push und PR macht der Nutzer.
- **Claude committet und pusht nicht.** Commits, Pushes und PRs macht ausschließlich der Nutzer. Änderungen nur im Arbeitsverzeichnis vorbereiten, am Ende kurz zusammenfassen und einen passenden Commit-Text vorschlagen. Ausnahme nur, wenn der Nutzer es in der jeweiligen Anfrage ausdrücklich verlangt.
- Conventional Commits mit Scope = Projekt, Beschreibung auf Deutsch: `docs(capulus-ansible): …`, `docs(capulus-operator): …`, `chore(deps): …`.
- `docs:`, `chore:` und `refactor:` lösen kein Release aus; `feat:` und `fix:` schon (semantic-release).
