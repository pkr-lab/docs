# Phase 5 — Strukturvorschlag für den Ausbau der Website

Vorschlag, noch nicht umgesetzt. An der Website wurde nichts umgebaut. Er setzt voraus, dass die Nachprüfung nach der Repo-Aufspaltung ([600b0](../capulus-core/6-hintergruende/600b0-nachpruefung-repo-aufspaltung.md)) abgeschlossen ist. Deren Live-Tests stehen noch aus.

Ausgangslage: 11 statische HTML-Seiten, JSON-LD nur in `index.html`, drei Seiten ohne Open Graph (`impressum.html`, `datenschutz.html`, `sitemap.html`), `sitemap.xml` ohne `lastmod`, minimale `robots.txt`, Update-Feed von Hand über `data/updates.json` und einen n8n-Workflow ([seo.md](seo.md)). Veröffentlicht wird per `git pull` auf dem netcup-Server; der Deploy-Workflow läuft derzeit nicht, weil die Repo-Secrets fehlen.

---

## Zielbild

| Bereich | Heute | Danach |
|---|---|---|
| Neue Seite `capulus.html` | – | Architekturdiagramm und Projektkarten, gespeist aus `data/projects.json` und `data/releases.json` |
| Release-Daten | manuell | Workflow erzeugt `data/releases.json` aus GitHub Releases |
| SEO | von Hand gepflegt | erzeugt: JSON-LD, Open Graph, `lastmod`, `robots.txt`, Sitemap |
| Inhalt, Doku | manuell | bleibt manuell |
| Interne Hostnamen | in 3 Zeilen sichtbar | entfernt |

---

## 1. Seite `capulus.html`

Eine zusätzliche Seite im bestehenden Aufbau (gleiche Kopfzeile, Fußzeile, CSS und Icons wie `projekt.html`). `projekt.html` bleibt die Erklärseite zu capulus-core, `capulus.html` zeigt das Gesamtbild.

Abschnitte:

1. **Architekturdiagramm** als inline-SVG (kein Bild, skaliert, passt zum Farbschema):
   - Ebene Repos: `capulus-core` (GitOps, ArgoCD-Apps), `capulus-ansible` (Hosts, k3s, Semaphore), `capulus-ios` (App), `capulus-wiki-sync` (Docs nach Wiki.js).
   - Verbindungen: Ansible provisioniert die Cluster und rendert `argocd/bootstrap/`; ArgoCD liest `capulus-core`; die iOS-App ruft `carplay-api` im Cluster; wiki-sync liest `pkr-lab-docs` und schreibt nach Wiki.js.
   - Ebene Tool-Stack: k3s, ArgoCD, Tailscale, Cloudflare Tunnel, Sealed Secrets, Renovate und semantic-release, Monitoring.
   - Keine internen Hostnamen, IPs oder Namespaces im Diagramm.
2. **Projektkarten** aus `data/projects.json`: Name, Beschreibung, Sprache/Stack, aktuelle Version (aus `releases.json`), Link auf GitHub und Doku.
3. **Neueste Releases** als kurze Liste aus `data/releases.json`.

Daten und Darstellung bleiben getrennt wie bei `apps.json` und `js/apps.js`: neues `js/capulus.js` lädt beide JSON-Dateien und rendert die Karten. Die Seite braucht dadurch keine Pflege bei einem Release.

`data/projects.json` **existiert noch nicht** (im Repo liegen nur `apps.json` und `updates.json`) und wird neu angelegt. Schema, handgepflegt:

```json
[
  {
    "repo": "capulus-core",
    "name": "capulus-core",
    "summary": "GitOps-Repository: ArgoCD-Apps, Promotion ENTW → TECH → PROD",
    "stack": ["Helm", "ArgoCD", "Go", "TypeScript"],
    "docs": "https://github.com/pkr-lab/pkr-lab-docs/tree/main/capulus-core",
    "order": 1
  }
]
```

Aufnehmen: capulus-core, capulus-ansible, capulus-ios, capulus-wiki-sync, cubi, ui-terminal-config. Die PowerShell-Repos und die Profil-Repos nur auf Wunsch.

---

## 2. Release-Feed `data/releases.json`

Neuer Workflow `.github/workflows/releases.yml` im Repo EDV-Kretzer:

- Trigger: täglich per `schedule`, `workflow_dispatch` und optional `repository_dispatch` (die Quell-Repos könnten nach ihrem Release ein Ereignis senden; ohne App oder PAT geht das nicht, daher zunächst nur Zeitplan).
- Quelle: `gh api repos/pkr-lab/<repo>/releases/latest` und die letzten 5 Releases je Repo aus `projects.json`. Öffentliche API, `GITHUB_TOKEN` genügt.
- Ausgabe: `data/releases.json` mit `repo`, `tag`, `name`, `published_at`, `url` und den ersten Zeilen der Release-Notes (Markdown entfernt).
- Schreiben: Bei Änderung ein Commit `chore(releases): Release-Daten aktualisiert [skip ci]`. Weil der Server per `git pull` zieht, müssen erzeugte Dateien im Repo liegen.
- Das Skript liegt als `scripts/build-releases.mjs` im Repo (Node ist in den Actions vorhanden), damit es lokal prüfbar ist.

Zu klären: Der Bot-Commit auf `main` funktioniert nur, solange keine Rulesets aktiv sind (siehe [600b0](../capulus-core/6-hintergruende/600b0-nachpruefung-repo-aufspaltung.md#rulesets)). Alternative ohne Commit: Der Workflow baut die Dateien und deployt sie per `rsync`/SSH, dann müssten sie nicht im Repo liegen.

---

## 3. Automatisierte SEO

Ein Skript `scripts/build-seo.mjs`, ausgeführt im selben Workflow wie die Release-Daten und in der CI jedes PR (dort als reine Prüfung ohne Schreiben):

| Ergebnis | Quelle |
|---|---|
| `sitemap.xml` mit `lastmod` | Datum des letzten Git-Commits je Datei; `priority` und `changefreq` aus einer kleinen Tabelle im Skript |
| `robots.txt` | Vorlage: `Allow: /`, Sitemap-Verweis, `Disallow:` für `/data/` und `/automation/` |
| Open Graph und Twitter-Card | aus `<title>` und `<meta name="description">` der Seite, Bild aus einer Standard-Vorgabe; Impressum, Datenschutz und Sitemap-Seite erhalten sie ebenfalls (Frage 2) |
| JSON-LD | `Person` und `WebSite` auf der Startseite, `BreadcrumbList` auf Unterseiten, `SoftwareSourceCode` je Projekt auf `capulus.html`, `ItemList` auf `anwendungen.html` aus `apps.json` |
| Prüfungen (Fehler in der CI) | Titel- und Description-Länge, ein Canonical je Seite, jeder interne Link und jedes Bild existiert, jede Seite steht in der Sitemap |

Technik: Die erzeugten Blöcke stehen zwischen Markern (`<!-- seo:start -->` … `<!-- seo:end -->`) und werden vom Skript ersetzt, so bleibt der übrige HTML-Code von Hand pflegbar und ein erneuter Lauf ändert nichts, wenn nichts zu ändern ist. Inhalt, Texte und Doku bleiben manuell.

Die Doku [seo.md](seo.md) wird zur Beschreibung des Skripts statt einer Liste von Hand gesetzter Werte.

---

## 4. Bereinigung

| Stelle | Änderung |
|---|---|
| `projekt.html:182` | Interne Zeile `nextcloud.prod.homeserver` samt Kommentarzeile „intern · LAN & Tailnet“ streichen, nur die externe Zeile `nextcloud-prod.pke-lab.de` bleibt |
| `projekt.html:88` | Punkt „Split-DNS via dnsmasq + Pi-hole für `*.homeserver`“ umformulieren zu „Split-DNS für interne Dienste“ |
| `data/apps.json:16` | Beschreibung von Pi-hole: „… Split-DNS für interne Dienste.“ statt `*.homeserver` |
| `data/apps.json:21-28` | Eintrag `authentik` streichen (laut [Incident 030](../capulus-core/7-security-incidents/030-minio-oidc-secret-in-git-historie.md) entfernt) |
| `data/updates.json` | kein Eintrag nennt Authentik oder Hostnamen, nichts zu tun |

Die drei Hostnamen-Stellen sind eine Einschränkung im Sinne von [Incident 034](../capulus-core/7-security-incidents/034-oeffentliches-repo-legt-interne-topologie-offen.md): Das Repo ist öffentlich, die Seite ebenfalls.

---

## Reihenfolge und Aufwand

1. Bereinigung (4), kleine Änderung, sofort.
2. SEO-Skript (3) und CI-Prüfung, danach läuft jede weitere Änderung geprüft.
3. `projects.json`, Release-Workflow (2).
4. `capulus.html` mit SVG und JS (1).
5. Deploy reparieren (Secrets setzen oder auf `rsync` umstellen).

---

## Fragen an dich

1. Bot-Commits auf `main` (Variante 1) oder Deploy per `rsync` (Variante 2)?
2. Sollen Impressum, Datenschutz und Sitemap-Seite in die Suchergebnisse (`index`) oder `noindex` bleiben?
3. Welche Repos gehören in die Projektkarten?
4. Bleibt der n8n-Workflow für `updates.json` bestehen, oder sollen Releases die Neuigkeiten ersetzen?
