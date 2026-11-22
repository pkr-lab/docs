# NINA-Warnlage — Katastrophenschutz-Dashboard

> **Cluster:** TECH · Ordner `argocd/apps/tech/monitoring/` (Dashboard in Grafana) · URL `https://grafana.tech.homeserver`. `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

Das Grafana-Dashboard **„NINA Warnlage — Katastrophenschutz“** zeigt die aktuell aktiven amtlichen Warnmeldungen für einen Landkreis. Gedacht ist es für die DLRG und
den Katastrophenschutz in Andernach (Landkreis Mayen-Koblenz). Es ersetzt das frühere statische DWD-Linkpanel im [Rhein-Dashboard](30090-rhein-dashboard.md).

Grafana URL: **https://grafana.tech.homeserver** → Ordner *OG Andernach* → *NINA Warnlage — Katastrophenschutz* (UID `nina-warnlage`).

---

## Datenquelle

| Quelle | Betreiber | Zweck |
|---|---|---|
| **NINA-Warn-API** (`https://warnung.bund.de/api31/dashboard/<ARS>.json`) | BBK | Aktive Warnungen je Gebiet aus MoWaS, Katwarn, BiWAPP, DWD, Hochwasserzentralen und Polizei |

Die API ist öffentlich und braucht keinen Schlüssel. Das Gebiet wird über den Amtlichen Regionalschlüssel (ARS) des Landkreises gewählt. NINA cached die Antwort etwa
5 Minuten, das Dashboard aktualisiert sich alle 5 Minuten. Der Host `https://warnung.bund.de` steht in den `allowedHosts` der Infinity-Datasource
(`templates/datasource-infinity.yaml`).

---

## Aufbau

| Panel | Inhalt |
|---|---|
| Aktive Warnungen | Anzahl gesamt und je Stufe (Extrem, Schwer, Markant, Gering), ohne Warnung grün |
| Warnungen | Tabelle mit Zeit, Stufe, Quelle, Art und Meldungstitel, neueste zuerst. Ohne Warnung steht dort „Keine aktiven Warnungen für diese Region“ |
| Warnstufen und Quellen | Erklärung der vier Stufen und der NINA-Quellen |
| Links | NINA, DWD-Warnkarte, Hochwasser RLP, Katwarn |

Die Variable **Region** (Standard Mayen-Koblenz) schaltet das Gebiet um:

| Region | ARS |
|---|---|
| Mayen-Koblenz | `071370000000` |
| Koblenz | `071110000000` |
| Neuwied | `071380000000` |
| Ahrweiler | `071310000000` |
| Cochem-Zell | `071350000000` |
| Rhein-Lahn | `071410000000` |
| Bonn | `053140000000` |
| Rhein-Sieg | `053820000000` |

Ein weiteres Gebiet kommt als Eintrag in die Variable `kreis` in `templates/dashboard-nina-warnlage.yaml`. Den ARS-Schlüssel prüft man mit:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "https://warnung.bund.de/api31/dashboard/<ARS>.json"
```

`200` heißt, der Schlüssel ist gültig. Eine leere Liste (`[]`) heißt nur, dass gerade nichts vorliegt.

---

## Warnstufen

| Stufe | NINA-Wert | Farbe |
|---|---|---|
| 1 · Gering | `Minor` | Gelb |
| 2 · Markant | `Moderate` | Orange |
| 3 · Schwer | `Severe` | Rot |
| 4 · Extrem | `Extreme` | Violett |

---

## Fehlerbehebung

| Symptom | Ursache und Abhilfe |
|---|---|
| Tabelle leer, Kacheln stehen auf 0 | Meistens richtig: Für die Region liegt nichts vor. Gegenprobe: `curl -s "https://warnung.bund.de/api31/dashboard/071370000000.json"` liefert `[]` |
| Fehler „host not allowed“ | `https://warnung.bund.de` fehlt in `allowedHosts`, nach Änderung Grafana neu starten (`kubectl -n monitoring rollout restart deploy/monitoring-grafana`) |
| Panel zeigt „No data“ trotz Warnungen | Antwortstruktur der API prüfen, die Abfrage nutzt `payload.data.severity`, `payload.data.provider`, `payload.data.msgType`, `i18nTitle.de` und `sent` |

Hintergrund zu den Infinity-Abfragen (Parser, JSONata): [Monitoring und Alerting](../6-hintergruende/60060-monitoring-und-alerting.md).
