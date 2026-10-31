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

## `data/projects.json`

Handgepflegte Liste der Projekte für die Karten auf `capulus.html`, in der Reihenfolge der Anzeige.

| Feld | Typ | Bedeutung |
|---|---|---|
| `repo` | String | Name des Repos unter `pkr-lab` |
| `name` | String | Anzeigename |
| `summary` | String | Kurzbeschreibung |
| `stack` | Liste | Technologien, das erste Element erscheint als `programmingLanguage` im JSON-LD |
| `docs` | String | Link zur Doku in pkr-lab-docs |

## `data/versions.json`

Wird nicht von Hand gepflegt. `scripts/build-releases.mjs` schreibt je Repo aus `projects.json` das neueste GitHub-Release (`tag`, `date`, `url`), der Workflow `releases.yml` stündlich. Repos ohne Release fehlen.

## `data/releases.json`

Die letzten zehn Releases aller capulus-Repos (jedes Repo aus `data/projects.json`, dessen Name mit `capulus-` beginnt, zusammen nach Veröffentlichungsdatum sortiert) für den Verlauf auf `updates.html`, neueste zuerst. Die Datei wird nicht von Hand gepflegt: `scripts/build-releases.mjs` erzeugt sie aus der GitHub-API (nur capulus-Repos, keine Entwürfe und Vorabversionen), der Workflow `release-verlauf` (`.github/workflows/releases.yml`) läuft stündlich, committet bei Änderung und deployt. Der Titel und der Text sind HTML-escaped, die Kurzfassung entsteht aus den Abschnitten „Features“ und „Bug Fixes“ der Release-Notes (je bis zu drei Einträge). Zusätzlich zu den folgenden Feldern gibt es `url` (Link auf das GitHub-Release).

`data/updates.json` ist die frühere, handgepflegte Liste. Sie wird von der Seite nicht mehr gelesen.

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

`kontakt.html` zeigt ein eigenes Formular mit E-Mail-Bestätigung (`js/kontakt-verifizierung.js`). Der Ablauf und die API stehen in [Komponenten](komponenten.md#kontaktformular-mit-e-mail-bestätigung), die Server-Seite in [ticket-gate](../capulus/capulus-core/3-apps-workloads/300r0-ticket-gate.md).

| Einstellung | Wo | Bedeutung |
|---|---|---|
| `API_BASE` | `js/kontakt-verifizierung.js` (Base64) | Adresse von ticket-gate, aktuell `https://ticket-gate-tech.pke-lab.de` |
| erlaubte Origins | `ALLOWED_ORIGINS` in ticket-gate | die Website-Hosts (`www.edv-kretzer.de`, `edv-kretzer.de`); lokale Tests von `localhost` aus brauchen einen Eintrag |
| Gültigkeit und Limits | `config:` in ticket-gate | 10 min Code, 5 Versuche, 30 min Token, 1 Mail pro 60 s |

Alternativ steht die E-Mail-Adresse im Impressum.

## n8n-Automatisierung

`automation/n8n-update-veroeffentlichen.json` ist ein n8n-Workflow für die frühere Liste `data/updates.json`, die `updates.html` nicht mehr anzeigt. Er ist damit ohne Wirkung auf die Seite und kann in n8n deaktiviert werden. Bei einem GitHub-Release von capulus-core oder per Webhook liest er `data/updates.json` über die GitHub-API, hängt einen Eintrag an und committet die Datei zurück. Nach dem Commit deployt `deploy.yml`.


---

## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| Katalog leer nach Änderung an `apps.json` | JSON-Syntaxfehler | `python3 -m json.tool data/apps.json` |
| App erscheint nur unter „Alle“ | Kategorie nicht in `CATEGORIES` von `js/apps.js` | Kategorie ergänzen oder bestehende wählen |
| Release-Verlauf alt | `releases.yml` lief nicht oder `releases.json` nicht deployt | [Automatisierung: Fehlersuche](automatisierung.md#debugging-und-fehlersuche) |
| Status-Widget leer | Slug `homelab` oder CORS | [Komponenten: Fehlersuche](komponenten.md#debugging-und-fehlersuche) |
| Banner erscheint nicht | `data/status.json`, `until` | [Komponenten](komponenten.md#status-hinweis-banner) |
| Kontaktformular fehlt | JavaScript aus oder `js/kontakt-verifizierung.js` nicht geladen | [Komponenten](komponenten.md#kontaktformular-mit-e-mail-bestätigung) |
