# INC-012 – Prüfwerkzeuge: teils installiert, CVE-Abgleich und SMART-Status weiter offen

| Feld | Wert |
|---|---|
| ID | INC-012 |
| Datum | 2026-11-14 |
| Kategorie | schwachstellen / hardware |
| Schweregrad | **Info** (Prüflücke, kein Fund) |
| Betroffen | Workstation und Projekte |

## Update 2026-11-14 (nach Installation durch den Benutzer)
Installiert und verfügbar: `smartctl`, `sensors`, `tpm2_getcap`, `lynis`, `rkhunter`, `chkrootkit`, `clamscan`, `pipx`, `go` (1.26.8). Weiterhin offen: `pip-audit`, `govulncheck`, `osv-scanner`, `trivy` (Upstream-Downloads, noch nicht freigegeben).

Ergebnisse ohne root:
- `sensors`: CPU Tctl 46,9 °C, NVMe 30,9 °C (Limit 87,8 °C), DIMMs 34 °C, GPU 41 °C: unauffällig. Lüfterdrehzahlen werden nicht ausgegeben.
- `clamscan` über `~/git`, `~/.ssh`, `~/.config/autostart`, `~/Downloads`: 2.835 Dateien, **0 infiziert** (Signaturen vom 14.11.2026, Engine 1.5.4). Kein Vollscan von `/`.
- `lynis audit system` (ohne root, eingeschränkt): Hardening-Index **66**, 249 Tests, 0 Warnungen, 38 Vorschläge (siehe INC-014).
- `smartctl -a /dev/nvme0` und `tpm2_getcap` ohne root nicht möglich; vom Benutzer mit `sudo` ausgeführt (Ausgabe übergeben):
  - **SSD Samsung 9100 PRO 4TB** (Firmware 0B2QNXH7): SMART *PASSED*, Critical Warning 0x00, Percentage Used **0 %**, Available Spare 100 %, Media/Data-Integrity-Errors **0**, Error-Log leer, Temperatur 31 °C, 364 Betriebsstunden, 19,0 TB geschrieben. Auffällig nur: **22 Unsafe Shutdowns** bei 586 Power-Cycles (Stromausfälle/Hard-Resets/Absturz; nicht kritisch, aber Dateisystem-Check bei Gelegenheit). Kein Self-Test protokolliert: `sudo smartctl -t short /dev/nvme0` wäre möglich (schreibt nichts).
  - **TPM:** 2.0, AMD-fTPM (siehe INC-003).

Noch offen mit `sudo`: `rkhunter --check --sk --rwo`, `chkrootkit`, `lynis audit system` (Vollumfang), `cat /etc/teamviewer/global.conf` (Schlüsselnamen).

## Beschreibung und Nachweis
Vor der Installation fehlende Tools (`command -v` → MISSING): `lynis`, `rkhunter`, `chkrootkit`, `clamscan`, `trivy`, `pip-audit`, `osv-scanner`, `smartctl` (Paket `smartmontools`), `sensors` (Paket `lm-sensors`), `tpm2_getcap` (Paket `tpm2-tools`), `govulncheck`/`go`. Vorhanden: npm, lshw, dmidecode, fwupdmgr, mokutil, boltctl, nmap, restic, kubectl, kubeseal.

Folgen für die Prüfung:
- Abhängigkeiten: `capulus-operator/go.mod` (controller-runtime v0.25.2, client-go v0.37.1, `go 1.26.0`) und die Python-Basis `python:3.14-slim` (wiki-sync, keine Requirements-Datei gefunden) wurden **nicht** gegen CVE-Datenbanken geprüft. Es gibt keine `package.json`.
- Dockerfiles: positiv sind Nicht-root-User (`65532`, `1000`) und Distroless-Image; negativ sind Basis-Images ohne Digest-Pinning (`golang:1.26-alpine`, `python:3.14-slim`, `distroless/static-debian12:nonroot`). Wiki-sync hat `renovate.json`.
- SSD-Gesundheit der Samsung SSD 9100 PRO 4TB (reallokierte Sektoren, Wear) nicht gelesen; fwupd meldet für das Laufwerk kein Firmware-Update.
- `boltctl list` und `/sys/class/iommu`: Thunderbolt-Liste leer; `/sys/class/iommu/ivhd0` und `AMD-Vi`-Meldungen vorhanden, d. h. IOMMU aktiv. Vollständige DMA-Protection-Aussage nicht getroffen.

## Risiko
Ohne Scanner bleiben bekannte CVEs in Go-Modulen und Container-Images, Rootkits sowie ein sterbendes Laufwerk unentdeckt.

## Empfohlener Fix
Freigabe erteilen für (alle aus den Ubuntu-Repos, reine Prüfwerkzeuge): `smartmontools`, `lm-sensors`, `tpm2-tools`, `lynis`, `rkhunter`, `chkrootkit`; außerdem `osv-scanner` und `trivy` (Upstream, nicht in Ubuntu-Repos; Installation per Release-Binary mit Prüfsumme). Danach lasse ich die Scans (nur lesend) laufen. `sudo smartctl -a /dev/nvme0` etc. brauchen root; ich erkläre den Befehl vorher.

## Fix-Skript
[`scripts/incident-fixes/INC-012-pruefwerkzeuge-nachinstallieren.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-012-pruefwerkzeuge-nachinstallieren.sh) – installiert ohne `sudo` in dein Home: `pip-audit` (per `pipx`), `govulncheck` und `osv-scanner` (per `go install`, Prüfsummen über die Go-Modul-Datenbank). `trivy` ist nicht enthalten (kein sauberer `go install`-Weg; Installation über das Aquasecurity-APT-Repository oder ein Release-Binary mit Prüfsumme, nur nach deiner Entscheidung). Danach Scans: `(cd capulus-operator && govulncheck ./...)`, `osv-scanner scan source -r ~/git/github/capulus`.

## Rollback
`pipx uninstall pip-audit`; `rm ~/go/bin/govulncheck ~/go/bin/osv-scanner`.

## Quellen
- https://google.github.io/osv-scanner/
- https://trivy.dev/
- https://cisofy.com/lynis/
