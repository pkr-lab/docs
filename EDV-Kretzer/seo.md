# SEO-Einstellungen

Stand der Suchmaschinen-Einstellungen der Website. Diese Seite wird in Phase 5 (Ausbau der Website) fortgeschrieben.

## `robots.txt`

```
User-agent: *
Allow: /

Sitemap: https://www.edv-kretzer.de/sitemap.xml
```

Alle Seiten sind erlaubt, der Verweis auf die Sitemap ist absolut.

## `sitemap.xml`

Enthält die 11 Seiten der Website mit `loc` und `priority` (Startseite 1.0, Projekt und Anwendungen 0.9, rechtliche Seiten 0.2). Ein `lastmod` fehlt noch.

## Meta-Angaben je Seite

| Angabe | Stand |
|---|---|
| `<title>` | auf allen 11 Seiten gesetzt, nach dem Muster „Thema – Zusatz \| EDV-Kretzer“ |
| `<meta name="description">` | auf allen Seiten vorhanden |
| `<meta name="robots" content="index, follow">` | auf allen Seiten |
| Canonical-Link | auf allen Seiten |
| Open Graph | vorhanden, außer auf `impressum.html`, `datenschutz.html` und `sitemap.html` |
| JSON-LD | nur auf `index.html` (`Person`) |
