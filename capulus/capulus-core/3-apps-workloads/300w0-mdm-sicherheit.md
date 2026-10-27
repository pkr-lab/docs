# MDM — Sicherheit

Welche Endpunkte öffentlich sind, wo die Geheimnisse liegen, wie die NetworkPolicies den Namespace abschotten und welche Risiken bleiben. Überblick: [300r0](300r0-mdm.md).

> **Kernaussage:** Ein MDM-Server ist ein Hochrisiko-Ziel. Wer die NanoMDM-API erreicht, kann **alle iPads aus der Ferne löschen** (`EraseDevice`), sperren, neue Konfigurationen und Apps verteilen und das Gerät mit einer Konfiguration übernehmen. Die Admin-API ist deshalb ausschließlich cluster-intern erreichbar.

---

## Exponierte Endpunkte

Öffentlich über den Cloudflare-Tunnel (Host `mdm-prod.pke-lab.de`) und intern (`mdm.prod.homeserver`) sind genau diese Pfade:

| Pfad | Zweck | Schutz |
|---|---|---|
| `/mdm` (exakt) | Check-in und Befehle der iPads | Gerät muss ein Zertifikat der SCEP-CA vorweisen (Signatur im `Mdm-Signature`-Header, NanoMDM prüft die Kette gegen `ca.pem`) |
| `/mdm/enroll` (exakt) | liefert das Enrollment-Profil aus | keiner: Das Profil enthält die SCEP-Challenge, siehe unten |
| `/scep` | Ausstellung der Geräte-Zertifikate | SCEP-Challenge |

Alle anderen Pfade (`/v1/…`, `/version`, `/migration`, `/authproxy`, `/proxy/…`) sind **nicht** im Ingress und liefern den 404 von Traefik. NanoMDM läuft ohne `-migration`, ohne `-auth-proxy-url` und ohne `-retro`. Prüfung von außen:

```bash
curl -s -o /dev/null -w '%{http_code}\n' https://mdm-prod.pke-lab.de/v1/pushcert
curl -s -o /dev/null -w '%{http_code}\n' https://mdm-prod.pke-lab.de/proxy/capulus/account
curl -s -o /dev/null -w '%{http_code}\n' https://mdm-prod.pke-lab.de/version
```

Alle drei müssen 404 liefern. Die Admin-APIs sind zusätzlich durch einen API-Schlüssel (HTTP Basic) geschützt, der nur im SealedSecret, in `credentials.env` und in den n8n-Credentials steht.

### Cloudflare

Das öffentliche TLS-Zertifikat stammt von Cloudflare (Wildcard `*.pke-lab.de`). Cloudflare beendet die TLS-Verbindung und kann damit Enrollment-Profil, SCEP-Challenge und MDM-Verkehr im Klartext sehen. Das gilt für alle öffentlichen Apps in diesem Cluster ([e0000](../e-externe-erreichbarkeit/e0000-cloudflare-tunnel.md)). Geräte-Zertifikate kommen deshalb nicht per TLS-Client-Zertifikat an, sondern als signierter Header.

---

## Secrets

Alles im Repo ist ein `SealedSecret`, nichts im Klartext. Die Klartexte liegen nur in `~/mdm-secrets/` auf dem Admin-Rechner (und im Passwortmanager). Aufstellung (ausführlich im `ops/token-register.yaml`):

| Geheimnis | Wo | Auswirkung bei Verlust oder Leck |
|---|---|---|
| NanoMDM-API-Schlüssel | SealedSecret `mdm-secrets`, n8n-Credential | Vollzugriff auf alle iPads, einschließlich Löschen |
| NanoDEP-API-Schlüssel | SealedSecret `mdm-secrets`, n8n-Credential | DEP-Konto über den Proxy ansprechbar, Profile zuweisen |
| SCEP-CA-Schlüssel | SealedSecret `mdm-scep-ca`, `~/mdm-secrets/scep-ca/`, Backup | Ausstellung beliebiger Geräte-Identitäten, danach Neu-Enrollment aller iPads |
| SCEP-Challenge | SealedSecret `mdm-secrets`, Enrollment-Profil | Fremde können Geräte-Zertifikate beziehen und Geräte beim MDM anmelden |
| Push-Zertifikat und Schlüssel | NanoMDM-Datenbank (nicht im Repo), Backup | Push an alle iPads |
| DEP-Token und -Schlüssel | NanoDEP-Datenbank (nicht im Repo), Backup | Abfrage und Zuweisung von Geräten bei Apple Business |
| Datenbank-Passwörter | SealedSecret `mdm-secrets` | Lesezugriff auf alle genannten Tabellen |
| restic-Passwort | SealedSecret `mdm-secrets`, Passwortmanager | ohne Passwort keine Wiederherstellung, mit Passwort und Volume Lesezugriff auf alles |
| Webhook-Token | SealedSecret `mdm-secrets`, Webhook-Pfade in n8n | Geräte-Events vortäuschen |
| Formular-Passwort | nur `credentials.env` und n8n-Credential | Push-Zertifikat und DEP-Token überschreiben |
| n8n-API-Schlüssel (Import) | wird nie gespeichert, nur als Umgebungsvariable | — |

Das Push-Zertifikat und der DEP-Token gehören bewusst in die Datenbank statt in SealedSecrets, weil NanoMDM sie per API entgegennimmt und beim Verlängern ersetzt.

Der Datenbankbenutzer für n8n (`mdmauto`) liest nur: Geräte-ID, Seriennummer, Enrollment-Typ und Aktivstatus sowie `topic`, `cert_pem` und `updated_at` des Push-Zertifikats (kein Schlüssel). Schreiben darf er nur im Schema `automation`.

---

## NetworkPolicies

Im Namespace gilt Default-Deny für Ingress und Egress. Freigaben (`templates/networkpolicy.yaml`):

| Policy | Erlaubt |
|---|---|
| `allow-dns-egress` | alle Pods → DNS (kube-system, Port 53) |
| `ingress-device-endpoints` | Traefik (kube-system) → `nanomdm`, `scep`, `enroll` |
| `ingress-admin-api-from-n8n` | Namespace `n8n` → `nanomdm:9000`, `depserver:9001` |
| `ingress-depserver-from-depsyncer` | `depsyncer` → `depserver` |
| `ingress-postgres` | `nanomdm`, `depserver`, `depsyncer`, `backup` und Namespace `n8n` → Postgres 5432 |
| `egress-postgres` | `nanomdm`, `depserver`, `depsyncer`, `backup` → Postgres |
| `egress-internet` | `nanomdm`, `depserver`, `depsyncer`, `backup` → öffentliche Adressen auf Port 443 (APNs, Apple Business), private Netze ausgenommen |
| `egress-backup-packages` | `backup` → Port 80 (Alpine-Pakete) |
| `egress-webhook-to-n8n` | `nanomdm`, `depsyncer` → n8n 5678 |
| `egress-depsyncer-to-depserver` | `depsyncer` → `depserver` |

`scep`, `enroll` und `postgres` dürfen **nichts** nach außen. Zusätzlich legt Ansible `tier-default-ingress` an (Namespace `mdm`: gleicher Namespace, kube-system, monitoring, cloudflared, n8n; Namespace `n8n`: zusätzlich `mdm` für die Webhooks). Konzept: [d0030](../d-sicherheit/d0030-network-policies.md).

---

## Weitere Schutzmaßnahmen

- Pods laufen als Nicht-Root mit `readOnlyRootFilesystem`, ohne Capabilities, mit `RuntimeDefault`-Seccomp-Profil und ohne ServiceAccount-Token (Ausnahme: der Backup-Job als Root für die Paketinstallation, ohne Netzwerkzugriff außer Postgres und Paketquellen).
- Alle Images sind auf Tag und Digest gepinnt.
- Die n8n-Formulare sind per Basic Auth geschützt. n8n selbst ist nicht öffentlich erreichbar ([30070](30070-n8n.md#sicherheit)).
- Der Push-Zertifikat-Upload prüft, dass das Topic unverändert bleibt. Dadurch kann ein falscher Upload nicht versehentlich alle iPads verwaisen.

---

## Risiken und Restpunkte

| Risiko | Einschätzung und Maßnahme |
|---|---|
| Kompromittierter MDM-Server oder Zugriff auf die NanoMDM-API | Kann **alle iPads löschen**, sperren oder umkonfigurieren. API nur im Cluster, Schlüssel nur im SealedSecret und im Passwortmanager, NetworkPolicy lässt nur n8n zu. Kein Zugriff über `kubectl` für Unbefugte; wer den Namespace `mdm` lesen kann, liest die Schlüssel |
| SCEP-Challenge im Enrollment-Profil | Das Profil wird ohne Anmeldung ausgeliefert (nur so funktioniert der automatische Abruf). Wer die URL kennt, kann sich ein Geräte-Zertifikat holen und ein fremdes Gerät beim MDM registrieren. Das Gerät bekäme keine Konfiguration von uns, aber Befehle an unbekannte Geräte lassen sich nur über die Datenbank beobachten. Challenge nach Verdacht rotieren ([300v0](300v0-mdm-betrieb.md#scep-challenge-rotieren)) |
| SCEP-Server ohne Härtungen | Das Projekt `micromdm/scep` nennt den Server selbst einfach und empfiehlt für robustere Lösungen andere Produkte. Er ist hier nur über die Challenge geschützt und stellt Zertifikate ohne weitere Prüfung aus |
| Push-Zertifikat über Drittanbieter (mdmcert.download) | Der Dienst signiert die Anfrage und sieht dabei die Anfrage, nicht den privaten Schlüssel. Er ist laut Betreiber nur für Test und Entwicklung gedacht. Ist er nicht mehr erreichbar, fehlt der Weg zur Verlängerung. Dann bleibt der direkte Weg über einen eigenen MDM-Vendor-Zertifikatsprozess bei Apple ([600d0](../6-hintergruende/600d0-mdm.md#push-zertifikat-ohne-eigene-vendor-cert)) |
| Cloudflare sieht den Verkehr | Siehe oben. Die API-Pfade sind nicht darüber erreichbar |
| Geräte-Zertifikate 10 Jahre gültig | Kein Ablauf-Risiko im Gerätelebenszyklus, aber ein abgeflossenes Zertifikat bleibt lange gültig. Sperren geht nur über Austausch der CA ([300v0](300v0-mdm-betrieb.md#scep-ca-erneuern)) |
| Backup enthält alle Schlüssel | Das restic-Repository liegt verschlüsselt auf dem NAS. Wer das Passwort und das Volume hat, liest alles |
| Menschlicher Fehler beim Verlängern | „Create“ statt „Renew“ erzeugt ein neues Topic. Das Formular lehnt es ab, die Checkliste warnt davor |
| n8n-API-Schlüssel im Chat oder Terminal | Der Schlüssel für den Workflow-Import ist ein mächtiger Schlüssel auf n8n. Nach dem Import in n8n widerrufen und bei Bedarf neu erzeugen |
