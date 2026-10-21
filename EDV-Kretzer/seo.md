# SEO-Einstellungen

Die Suchmaschinen-Angaben entstehen automatisch. `scripts/build-seo.mjs` schreibt sie in die Seiten und erzeugt `sitemap.xml` und `robots.txt`. Von Hand gepflegt werden nur Inhalt, `<title>`, `<meta name="description">` und die Canonical-Adresse jeder Seite.

## Was das Skript erzeugt

| Ergebnis | Quelle |
|---|---|
| Open Graph (`og:type`, `og:site_name`, `og:title`, `og:description`, `og:url`, `og:locale`, `og:image`) und Twitter-Card | Titel (ohne „\| EDV-Kretzer“) und Description der Seite, Vorschaubild je Seite aus einer Tabelle im Skript |
| JSON-LD (`@graph`) | Startseite: `Person`, `WebSite`, `WebPage`. Unterseiten: `WebPage` und `BreadcrumbList`. `capulus.html`: zusätzlich `ItemList` mit `SoftwareSourceCode` je Projekt (Version aus `data/versions.json`). `anwendungen.html`: `ItemList` aus `data/apps.json` |
| `sitemap.xml` | alle Seiten mit kanonischer Adresse, `lastmod` (Datum des letzten Git-Commits der Seite und ihrer Datendateien), `changefreq` und `priority` |
| `robots.txt` | `Allow: /`, `Disallow` für `/automation/`, `/scripts/` und `/.github/`, Verweis auf die Sitemap |

Die erzeugten Tags tragen das Attribut `data-seo`. Bei jedem Lauf entfernt das Skript alle Open-Graph-, Twitter- und JSON-LD-Angaben und schreibt sie vor `</head>` neu. Ein zweiter Lauf ändert nichts (idempotent). Manuell gesetzte Open-Graph- oder JSON-LD-Tags werden dabei überschrieben.

## Prüfungen

`node scripts/build-seo.mjs --check` schreibt nichts und liefert einen Fehlercode bei:

- fehlendem `<title>`, fehlender Description, keinem oder mehr als einem Canonical, Canonical ungleich `https://www.edv-kretzer.de/<Datei>` (Startseite: `/`)
- internen `href`/`src` ohne Zieldatei
- fehlendem Vorschaubild
- einer HTML-Datei, die nicht in der Tabelle `PAGES` im Skript steht (und umgekehrt)

Warnungen (kein Fehler) gibt es bei Titeln über 80 Zeichen und Descriptions unter 50 oder über 220 Zeichen. Die CI (`ci.yml`, „Seiten-Prüfung“) führt die Prüfung auf jedem Pull Request aus.

| Angabe | Stand |
|---|---|
| `<title>` | auf allen 11 Seiten gesetzt, nach dem Muster „Thema – Zusatz \| EDV-Kretzer“ |
| `<meta name="description">` | auf allen Seiten vorhanden |
| `<meta name="robots" content="index, follow">` | auf allen Seiten |
| Canonical-Link | auf allen Seiten |
| Open Graph | vorhanden, außer auf `impressum.html`, `datenschutz.html` und `sitemap.html` |
| JSON-LD | nur auf `index.html` (`Person`) |


---

## Fehlersuche

| Prüfung | Befehl oder Ort | Hinweis |
|---|---|---|
| `robots.txt` erreichbar | `curl -s https://www.edv-kretzer.de/robots.txt` | muss `Sitemap: https://www.edv-kretzer.de/sitemap.xml` enthalten |
| Sitemap vollständig | `grep -c '<loc>' sitemap.xml` (derzeit 11) | neue Seite in `sitemap.xml` und `sitemap.html` eintragen ([Nutzung](nutzung.md)) |
| Meta-Angaben je Seite | `<title>`, `description`, Canonical, Open Graph | Open Graph fehlt bewusst auf Impressum, Datenschutz und Sitemap |
| Seite wird nicht indexiert | Suchmaschinen-Werkzeug des Anbieters | `robots.txt` und `<meta name="robots">` prüfen |
| Alte Seite im Index | `lastmod` fehlt in der Sitemap | [Phase 5](phase-5-strukturvorschlag.md) |

Allgemeines: [Debugging und Fehlersuche](debugging.md).
