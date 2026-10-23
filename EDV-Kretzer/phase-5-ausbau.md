# Phase 5 — Ausbau der Website (umgesetzt)

Stand 2026-10-19. Der frühere Strukturvorschlag ist bis auf die unten genannten Abweichungen umgesetzt. Ausgangslage war: 11 statische HTML-Seiten, JSON-LD nur in `index.html`, drei Seiten ohne Open Graph, `sitemap.xml` ohne `lastmod`, minimale `robots.txt`, Update-Feed von Hand über `data/updates.json` und einen n8n-Workflow.

## Ergebnis

| Bereich | Umsetzung |
|---|---|
| Seite `capulus.html` | SVG-Architekturdiagramm (Repositories, Cluster, Werkzeuge), Projektkarten aus `data/projects.json` mit Version aus `data/versions.json`, die fünf neuesten Releases aller capulus-Repos. Beschreibung in [index.md](index.md) und [konfiguration.md](konfiguration.md) |
| Release-Daten | `scripts/build-releases.mjs` und der Workflow `releases.yml` (stündlich) erzeugen `data/releases.json` (nur capulus-core, zehn Einträge, für `updates.html`) und `data/versions.json` (je Projekt das neueste Release) |
| SEO | `scripts/build-seo.mjs`, Workflow `seo.yml` und Prüfung in `ci.yml`, siehe [seo.md](seo.md) |
| Aufräumen | Interne `*.homeserver`-Hostnamen aus `projekt.html` und `data/apps.json` entfernt, Authentik aus `data/apps.json` gestrichen (laut [Incident 030](../capulus-core/7-security-incidents/030-minio-oidc-secret-in-git-historie.md) entfernt), `data/updates.json` und der alte n8n-Workflow dazu gelöscht |
| Zusätzlich | Statusbanner mit n8n-Formular ([nutzung.md](nutzung.md#status-hinweis-wartung-störung)), Deployment über den Git-Spiegel ([installation.md](installation.md#deployment-auf-netcup)) |

## Abweichungen vom Vorschlag

- `robots.txt` sperrt `/data/` nicht, weil Suchmaschinen die JSON-Dateien zum Darstellen der Seiten brauchen. Gesperrt sind `/automation/`, `/scripts/` und `/.github/`.
- Die Release-Daten liegen in zwei Dateien statt einer: `releases.json` nur für capulus-core (Wunsch für `updates.html`), `versions.json` für die Karten.
- Es gibt keine `.htaccess` zum Sperren der Ordner, siehe Offene Punkte in [seo.md](seo.md#offene-punkte).
- Bot-Commits auf `main` statt Deployment per `rsync`: möglich, weil kein Ruleset aktiv ist. Werden die Rulesets in EDV-Kretzer eingeschaltet, brauchen die Workflows `releases.yml`, `seo.yml` und `status.yml` eine Bypass-Ausnahme (siehe [600b0](../capulus-core/6-hintergruende/600b0-nachpruefung-repo-aufspaltung.md#rulesets)).
