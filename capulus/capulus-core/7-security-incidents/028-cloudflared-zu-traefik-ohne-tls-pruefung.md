# 028 cloudflared → Traefik mit `noTLSVerify: true`

- Severity: Low
- Bereich: Netzwerk
- Betroffen: `argocd/apps/tech/cloudflared/values.yaml:60`, `argocd/apps/prod/cloudflared/values.yaml`
- Status: Behoben am 2026-10-18 (im Code, gestaged; wirkt nach Merge und ArgoCD-Sync, Verifikation mit öffentlichem Aufruf nach dem Rollout)

## Beschreibung

Der Tunnel-Client verbindet sich mit `https://traefik.kube-system.svc.cluster.local:443` und prüft das Zertifikat nicht. Das Zertifikat stammt von der internen CA (SAN-Liste der `*.homeserver`-Hosts) und enthält den Service-Namen ohnehin nicht. Der Weg bleibt im Cluster; ein Pod, der den Service-Namen spoofen könnte, kann den Tunnelverkehr abgreifen.

## Nachweis

```
$ sed -n 57,60p argocd/apps/tech/cloudflared/values.yaml
      - hostname: "*.pke-lab.de"
        service: https://traefik.kube-system.svc.cluster.local:443
        originRequest:
          noTLSVerify: true
```

## Risiko / Angriffsszenario

Ein bösartiger Pod, der DNS/Service-Routing im Cluster verbiegt, kann TLS-terminierte Sitzungen aus dem Tunnel mitlesen; nur mit bereits vorhandener Cluster-Position relevant.

## Empfohlene Behebung

Root-CA als ConfigMap in den cloudflared-Pod einhängen und prüfen:
```diff
         originRequest:
-          noTLSVerify: true
+          caPool: /etc/cloudflared/ca/homeserver-root-ca.pem
+          originServerName: traefik.kube-system.svc.cluster.local
```
Das Wildcard-Zertifikat bzw. die SAN-Liste um den Service-Namen erweitern (cert-manager `Certificate`).

## Rollback

`noTLSVerify: true` wiederherstellen.

## Verifikation

```
kubectl -n cloudflared logs deploy/cloudflared | grep -i -E 'error|certificate'
curl -s -o /dev/null -w '%{http_code}\n' https://nextcloud-prod.pke-lab.de/status.php
```
Keine TLS-Fehler im Log, Tunnelverkehr liefert `200`.

## Risiko der Behebung

Niedrig.

## Behebung (2026-10-18)

**Fix.** `noTLSVerify: true` ist in TECH und PROD entfernt. cloudflared prüft das Traefik-Zertifikat jetzt gegen die Homeserver-Root-CA (`caPool: /etc/cloudflared/origin-ca.pem`, Datei aus der ConfigMap des Charts, Inhalt `tunnel.originCaPem` in `values.yaml`, identisch zu `pki/homeserver-root-ca.pem`). Statt die SAN-Liste um den Service-Namen zu erweitern (so empfohlen), setzt `originServerName` einen bereits enthaltenen Namen: TECH `homeserver`, PROD `nextcloud.prod.homeserver`. So braucht es keine Zertifikatsänderung und keine Reihenfolge zwischen Zertifikat-Erneuerung und Rollout. Geroutet wird weiter über den `Host`-Header, der Name wirkt nur auf SNI und Prüfung.

**Nachweis** (vor dem Merge):

| Test | Ergebnis |
|---|---|
| `openssl s_client -verify_hostname homeserver` gegen TECH-Traefik (`192.168.178.200`) mit nur der Root-CA | `Verification: OK` |
| `openssl s_client -verify_hostname nextcloud.prod.homeserver` gegen PROD-Traefik (`192.168.178.99`, Kette Blatt → PROD-Intermediate → Root) | `Verification: OK` |
| Go `tls.Dial` (nur Root-CA als `RootCAs`) | TECH mit `homeserver` OK, PROD mit `nextcloud.prod.homeserver` OK, TECH mit falschem Namen `FEHLER … certificate is valid for homeserver, …` (die Prüfung greift wirklich) |
| `cloudflared tunnel ingress validate` (Image `cloudflare/cloudflared:2026.9.3`) mit der gerenderten Konfiguration, beide Cluster | `OK`, Regel `*.pke-lab.de` passt |
| `helm template` | ConfigMap enthält `origin-ca.pem` (PEM identisch zur Datei in `docs/assets`) und die neuen `originRequest`-Felder |

**Geänderte Dateien.**

- `argocd/apps/tech/cloudflared/values.yaml`, `templates/configmap.yaml`, `argocd/apps/prod/cloudflared/values.yaml`, `templates/configmap.yaml`
- Doku: `capulus-core/6-hintergruende/60040-helm-charts-tech.md`, `capulus-core/e-externe-erreichbarkeit/e0000-cloudflare-tunnel.md`, `capulus-core/c-netzwerk-dns/c0040-domain-tiers.md`

**Rollback.** In beiden `values.yaml` `originRequest` auf `noTLSVerify: true` zurücksetzen (`git revert`), ArgoCD rollt die ConfigMap aus, die Pods starten durch die Checksum-Annotation neu.

**Offen.**

- Abhängigkeit: Der Name `nextcloud.prod.homeserver` muss in der PROD-SAN-Liste bleiben, siehe Hintergrunddoc.

## Nachtrag (2026-11-12): TECH-Tunnel prüft auch die PROD-Hosts

Seit dem Domainwechsel auf `pkr-lab.de` reicht der TECH-Tunnel die Hosts `nextcloud-prod`, `immich-prod`, `wiki-prod` und `xibo-prod` an den Traefik des PROD-Clusters (`https://192.168.178.99:443`) weiter. Die Regeln in `argocd/apps/tech/cloudflared/values.yaml` setzen dafür `caPool: /etc/cloudflared/origin-ca.pem` und `originServerName: nextcloud.prod.homeserver`, es gibt weiterhin kein `noTLSVerify`. Die Abhängigkeit von der SAN-Liste gilt damit für den **TECH-Tunnel** (vorher nur für den PROD-Tunnel).

| Voraussetzung | Wo |
|---|---|
| `nextcloud.prod.homeserver` steht in `dnsNames` des PROD-Zertifikats | [`certificate-homeserver-wildcard.yaml`](https://github.com/pkr-lab/capulus-core/blob/main/argocd/apps/prod/cert-manager/templates/certificate-homeserver-wildcard.yaml) |
| Die Root-CA in `tunnel.originCaPem` des TECH-Tunnels ist die aktuelle | `argocd/apps/tech/cloudflared/values.yaml`, identisch zu `pki/homeserver-root-ca.pem` |

**Prüfung** (am 2026-11-12 mit der CA aus der gerenderten ConfigMap gegen den PROD-Traefik: `Verification: OK`):

```
openssl s_client -connect 192.168.178.99:443 -servername nextcloud.prod.homeserver -CAfile pki/homeserver-root-ca.pem -verify_hostname nextcloud.prod.homeserver
```

**Fehlerbild.** Fehlt der Name in der SAN-Liste, liefern die vier PROD-Hosts 502, während der TECH-Tunnel `Ready` bleibt. Dann in den vier Regeln `originServerName` auf einen Namen setzen, der in der Liste steht.
