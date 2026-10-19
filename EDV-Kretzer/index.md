# EDV-Kretzer

Statische Website von EDV-Kretzer (<https://www.edv-kretzer.de>). Sie zeigt, was auf dem Capulus-Homelab läuft und wie es gebaut ist: das GitOps-Projekt capulus-core, den Katalog der selbst gehosteten Anwendungen, die iOS-App, Updates mit Live-Status sowie Kontakt, Impressum und Datenschutz.

Repository: [pkr-lab/EDV-Kretzer](https://github.com/pkr-lab/EDV-Kretzer)

## Überblick

Reines HTML, CSS und etwas JavaScript, ohne Build-Schritt und ohne Framework. Die Seite wird per `git pull` auf einem netcup-Webserver ausgeliefert.

| Seite | Inhalt |
|---|---|
| `index.html` | Startseite mit Kurzvorstellung |
| `projekt.html` | Das Projekt capulus-core: Architektur, Komponenten, Hintergründe |
| `capulus.html` | capulus im Überblick: SVG-Architekturdiagramm, Projektkarten mit aktueller Version, neueste Releases |
| `anwendungen.html` | Katalog der Self-Hosted-Dienste, geladen aus `data/apps.json` |
| `mediamtx-drohnenstream.html` | Anleitung: Drohnen-Livestream per MediaMTX zum ELW |
| `app.html` | Die eigene iOS-App fürs Homelab mit Screenshots |
| `updates.html` | Verlauf der letzten zehn capulus-core-Releases aus `data/releases.json` und Live-Status der Dienste |
| `ueber-mich.html`, `kontakt.html` | Vorstellung und Kontaktformular (Zammad) |
| `sitemap.html`, `sitemap.xml`, `robots.txt` | Seitenübersicht und Hinweise für Suchmaschinen |
| `impressum.html`, `datenschutz.html` | rechtliche Seiten |

## Architektur

```mermaid
flowchart LR
  browser["Browser"] --> html["HTML-Seiten"]
  html --> css["css/style.css, animations.css"]
  html --> js["js/main.js, apps.js, updates.js, capulus.js"]
  js -->|"fetch"| apps["data/apps.json"]
  js -->|"fetch"| upd["data/releases.json, versions.json, projects.json"]
  js -->|"fetch (öffentlich)"| kuma["Uptime-Kuma-Status-Page"]
  push["Push auf main"] --> gha["GitHub Action deploy.yml"]
  gha -->|"SSH: fetch und checkout"| netcup["netcup-Webserver"]
```

| Ordner / Datei | Inhalt |
|---|---|
| `assets/icons`, `assets/images` | Logo, Favicon, SVG-Sprite, Screenshots |
| `css/` | `style.css` (Designsystem mit CSS-Variablen), `animations.css` |
| `js/main.js` | Navigation, Dropdown, Einblend-Animationen, Kontaktformular-Einbindung |
| `js/apps.js` | rendert den Anwendungskatalog mit Filter und Suche |
| `js/updates.js` | rendert den Release-Verlauf und das Status-Widget |
| `js/capulus.js` | rendert Projektkarten und neueste Releases auf `capulus.html` |
| `data/` | `apps.json`, `projects.json`, `releases.json`, `versions.json`, `status.json` |
| `scripts/` | `build-releases.mjs` (Release-Daten), `build-seo.mjs` (SEO) |
| `automation/` | n8n-Workflow „Status-Hinweis schalten“ |
| `.github/workflows/` | `deploy.yml` (Deployment nach netcup), `releases.yml` (Release-Daten), `seo.yml` (SEO), `status.yml` (Status-Banner), `ci.yml` (Prüfung jedes PR) |

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Releases](releases.md)
- [SEO](seo.md)
- [Phase 5: Ausbau der Website](phase-5-ausbau.md)
