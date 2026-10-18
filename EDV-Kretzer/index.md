# EDV-Kretzer

Statische Website von EDV-Kretzer (<https://www.edv-kretzer.de>). Sie zeigt, was auf dem Capulus-Homelab läuft und wie es gebaut ist: das GitOps-Projekt capulus-core, den Katalog der selbst gehosteten Anwendungen, die iOS-App, Updates mit Live-Status sowie Kontakt, Impressum und Datenschutz.

Repository: [pkr-lab/EDV-Kretzer](https://github.com/pkr-lab/EDV-Kretzer)

## Überblick

Reines HTML, CSS und etwas JavaScript, ohne Build-Schritt und ohne Framework. Die Seite wird per `git pull` auf einem netcup-Webserver ausgeliefert.

| Seite | Inhalt |
|---|---|
| `index.html` | Startseite mit Kurzvorstellung und strukturierten Daten (`Person`) |
| `projekt.html` | Das Projekt capulus-core: Architektur, Komponenten, Hintergründe |
| `anwendungen.html` | Katalog der Self-Hosted-Dienste, geladen aus `data/apps.json` |
| `mediamtx-drohnenstream.html` | Anleitung: Drohnen-Livestream per MediaMTX zum ELW |
| `app.html` | Die eigene iOS-App fürs Homelab mit Screenshots |
| `updates.html` | Changelog aus `data/updates.json` und Live-Status der Dienste |
| `ueber-mich.html`, `kontakt.html` | Vorstellung und Kontaktformular (Zammad) |
| `sitemap.html`, `sitemap.xml`, `robots.txt` | Seitenübersicht und Hinweise für Suchmaschinen |
| `impressum.html`, `datenschutz.html` | rechtliche Seiten |

## Architektur

```mermaid
flowchart LR
  browser["Browser"] --> html["HTML-Seiten"]
  html --> css["css/style.css, animations.css"]
  html --> js["js/main.js, apps.js, updates.js"]
  js -->|"fetch"| apps["data/apps.json"]
  js -->|"fetch"| upd["data/updates.json"]
  js -->|"fetch (öffentlich)"| kuma["Uptime-Kuma-Status-Page"]
  push["Push auf main"] --> gha["GitHub Action deploy.yml"]
  gha -->|"SSH: git pull --ff-only"| netcup["netcup-Webserver"]
```

| Ordner / Datei | Inhalt |
|---|---|
| `assets/icons`, `assets/images` | Logo, Favicon, SVG-Sprite, Screenshots |
| `css/` | `style.css` (Designsystem mit CSS-Variablen), `animations.css` |
| `js/main.js` | Navigation, Dropdown, Einblend-Animationen, Kontaktformular-Einbindung |
| `js/apps.js` | rendert den Anwendungskatalog mit Filter und Suche |
| `js/updates.js` | rendert den Changelog und das Status-Widget |
| `data/` | `apps.json`, `updates.json` |
| `automation/` | n8n-Workflow zum Veröffentlichen von Updates |
| `.github/workflows/deploy.yml` | Deployment nach netcup |

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Releases](releases.md)
