# Konfiguration

## `data/apps.json`

Liste der Anwendungen für `anwendungen.html`.

| Feld | Typ | Bedeutung |
|---|---|---|
| `id` | String | eindeutige Kennung |
| `name` | String | Anzeigename |
| `icon` | String | Emoji als Symbol |
| `categories` | Liste | Kategorien für den Filter (`Infrastruktur`, `Monitoring`, `Sicherheit & Identity`, `Automatisierung`, `Kommunikation`, `Medien & Storage`, `Produktivität`) |
| `description` | String | Kurzbeschreibung |
| `tags` | Liste | Schlagwörter, durchsuchbar |
| `url` | String | Link zur Projektseite des Werkzeugs |

## `data/updates.json`

Liste der Einträge für den Changelog auf `updates.html`, neueste zuerst nach `date` sortiert.

| Feld | Bedeutung |
|---|---|
| `id` | eindeutige Kennung |
| `date` | Datum `JJJJ-MM-TT` |
| `tag` | Kategorie, z. B. `Feature` oder `Anwendungen` |
| `title`, `body` | Überschrift und Text |

## Status-Widget

`js/updates.js` liest die öffentliche Status-Page eines Uptime-Kuma über deren API (`/api/status-page/<slug>` und `/heartbeat/<slug>`) und zeigt je Dienst Online, Down, Prüfung läuft oder Wartung an. Die Basis-URL steht im Code kodiert, der Slug ist `homelab`. Es werden nur Daten angezeigt, es gibt keinen Link und kein iframe zum Dienst.

## Suchmaschinen

`robots.txt` erlaubt alle Seiten und verweist auf `sitemap.xml`. Die Einstellungen sind in [seo.md](seo.md) beschrieben.

## Kontaktformular

`kontakt.html` öffnet ein Zammad-Formular über `js/main.js`. Alternativ steht die E-Mail-Adresse im Impressum.

## n8n-Automatisierung

`automation/n8n-update-veroeffentlichen.json` ist ein n8n-Workflow. Bei einem GitHub-Release von capulus-core oder per Webhook liest er `data/updates.json` über die GitHub-API, hängt einen Eintrag an und committet die Datei zurück. Nach dem Commit deployt `deploy.yml`.
