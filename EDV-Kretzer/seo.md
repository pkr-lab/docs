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

## Ablauf

1. Push auf `main`: Der Workflow `seo.yml` führt das Skript aus (mit voller Git-Historie für `lastmod`), committet geänderte Dateien als `chore(seo): … [skip ci]` und startet das Deployment.
2. Neue Releases: `releases.yml` ruft das Skript mit auf, wenn sich `data/releases.json` oder `data/versions.json` ändern. Dann stehen Version und `lastmod` sofort richtig.

## Neue Seite anlegen

1. HTML-Datei anlegen mit `<title>`, Description und Canonical (`<link rel="canonical" href="https://www.edv-kretzer.de/<Datei>">`).
2. Eintrag in `PAGES` in `scripts/build-seo.mjs` ergänzen (Breadcrumb-Name, Priorität, Änderungshäufigkeit, optional Vorschaubild, Datendateien für `lastmod`, Gruppe).
3. In Navigation und `sitemap.html` verlinken. `sitemap.xml` entsteht beim nächsten Lauf.

## Offene Punkte

- Impressum, Datenschutz und `sitemap.html` stehen weiter im Index (`index, follow`). Wer sie ausschließen will, setzt `noindex` in der Seite und nimmt sie aus `PAGES`.
- Es gibt kein eigenes Vorschaubild für Open Graph, verwendet werden vorhandene Screenshots und das Foto. Seiten ohne Eintrag in der Tabelle haben kein `og:image`.
- `/data/` bleibt in `robots.txt` erlaubt, weil Suchmaschinen die JSON-Dateien zum Darstellen von `anwendungen.html` und `capulus.html` brauchen.
- `automation/`, `scripts/` und `.github/` liegen im Docroot und sind per URL abrufbar. Eine `.htaccess` mit `RedirectMatch 404` würde sie sperren, ist aber nicht eingerichtet: Erlaubt das Hosting `FileInfo` in `.htaccess` nicht, antwortet der Server überall mit 500. Vorher auf einem Testordner prüfen.
