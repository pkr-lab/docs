# Debugging und Fehlersuche

Zentrale Seite für die Website. Die Fehlerbilder der einzelnen Teile stehen bei den Komponenten ([Komponenten](komponenten.md#debugging-und-fehlersuche)) und bei der Automatisierung ([Automatisierung](automatisierung.md#debugging-und-fehlersuche)). Das projektübergreifende Kapitel ist die [zentrale Fehlersuche](../capulus/capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

---

## Wo man Fehler sieht

| Was | Wo |
|---|---|
| JavaScript-Fehler, fehlgeschlagene Abrufe | Entwicklertools des Browsers: Konsole und Netzwerk (F12). `apps.js` schreibt Ladefehler zusätzlich mit `console.error` |
| Deployment, Release-Verlauf, Status-Hinweis | GitHub → Repo EDV-Kretzer → Actions |
| Auslieferung auf dem Webserver | netcup-Kundenbereich bzw. Plesk, Zugriff per SSH auf das Hosting |
| Zammad, Status-Page | Cluster-Dienste, siehe [Zammad](../capulus/capulus-core/3-apps-workloads/30000-zammad.md) und [Uptime Kuma](../capulus/capulus-core/3-apps-workloads/30080-uptime-kuma.md) |

Die Website selbst schreibt keine Logs. Es gibt keinen Debug-Modus und kein Log-Level, nur die Konsole des Browsers.

---

## Lokal debuggen

```bash
git clone https://github.com/pkr-lab/EDV-Kretzer.git && cd EDV-Kretzer
python3 -m http.server 8000                 # http://localhost:8000/
python3 -m json.tool data/apps.json >/dev/null && echo ok      # JSON prüfen
node scripts/build-releases.mjs             # Release-Verlauf lokal erzeugen (Node 18 oder neuer, wegen fetch)
```

Die Seite muss über einen Webserver geöffnet werden, weil `fetch` auf `file://` scheitert. Für den Status-Banner lässt sich `data/status.json` lokal auf `"active": true` stellen. Die Seiten laden die Datei mit `no-store`, ein Neuladen genügt. Zammad und die Status-Page laufen auch lokal gegen die echten Dienste, `localhost` ist dort nicht freigegeben (CORS für die Status-Page erlaubt nur die Website-Hosts), das Widget zeigt lokal daher „Status gerade nicht abrufbar.“ (erwartet).

---

## Checkliste „Seite oder Feature funktioniert nicht“

1. **Statische Datei erreichbar?** `curl -I https://www.edv-kretzer.de/data/releases.json` liefert 200? Gleiches für `data/apps.json` und `data/status.json`.
2. **Konsole lesen:** rote Fehler zu `fetch`, CORS oder 404? Dann zuerst die Ursache dort.
3. **Stand auf dem Server:** Wurde der letzte Commit deployt? Actions → „Deploy to netcup“ grün? Der Lauf zeigt den ausgecheckten Commit nicht. Vergleich per Dateiänderung (`Last-Modified`).
4. **JSON gültig?** `python3 -m json.tool` auf die Datei. Ein Komma zu viel legt Katalog oder Verlauf lahm.
5. **Cache:** Strg+F5. `status.json` wird nie gecacht, `apps.json` und `releases.json` folgen den Browser-Standards.
6. **Externe Dienste:** Zammad (Chat, Formular) und Uptime Kuma (Status-Widget) laufen im Cluster. Sind sie aus, fehlen nur diese Teile ([Komponenten](komponenten.md)).
7. **Automatisierung:** Release-Verlauf bleibt alt oder Status-Hinweis erscheint nicht? → [Automatisierung](automatisierung.md#debugging-und-fehlersuche).

---

## Häufige Probleme

| Symptom | Ursache | Lösung |
|---|---|---|
| Alles leer, nur Rahmen | Seite per `file://` geöffnet | Webserver starten |
| Katalog, Verlauf oder Banner fehlen auf dem Server, lokal ok | Deployment nicht gelaufen oder fehlgeschlagen | Actions prüfen, [Installation](installation.md#deployment-auf-netcup) |
| Neue Seite fehlt in Navigation oder Sitemap | nicht eingetragen | [Nutzung](nutzung.md#inhalte-pflegen), [SEO](seo.md) |
| Chat und Formular fehlen | Zammad nicht erreichbar oder Skripte blockiert | Konsole, [Zammad](../capulus/capulus-core/3-apps-workloads/30000-zammad.md) |
| Status-Widget leer | Status-Page, Slug oder CORS | [Komponenten](komponenten.md#status-widget-jsupdatesjs) |
| `.github/` oder `automation/` im Browser abrufbar | liegen im Docroot (Deployment checkt das ganze Repo aus) | bei Bedarf per `.htaccess` sperren, siehe [Installation](installation.md#stolperstellen) |
| Suchmaschine zeigt veraltete Seiten | Sitemap ohne `lastmod`, keine Aktualisierung | [SEO](seo.md) |

Allgemeines Vorgehen: erst die Konsole des Browsers, dann die Actions, dann die Cluster-Dienste.
