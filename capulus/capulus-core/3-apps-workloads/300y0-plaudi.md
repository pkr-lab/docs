# Plaudi — Chat mit Claude

> **Cluster:** PROD · Ordner `argocd/apps/prod/plaudi/` · URL `https://plaudi-prod.pkr-lab.de` (extern, Cloudflare Tunnel), `plaudi.prod.homeserver` (intern). `kubectl`-Befehle für PROD laufen per `ssh ubuntu@192.168.178.99 "sudo k3s kubectl …"` ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

Selbst gehosteter Chat mit eigenem Verlauf je Chat, Anzeige von Nutzungslimit und Kontext und automatischem Wechsel zwischen mehreren API-Konten. Ein einzelner Node-Prozess mit SQLite, Quellcode und Entwicklerdoku im Repo [pkr-lab/chatbot](https://github.com/pkr-lab/chatbot). Hier steht nur der Betrieb im Cluster.

## Aufbau

```
Browser ──https──► Cloudflare ──► cloudflared (PROD) ──► Traefik ──► plaudi (Namespace plaudi, Port 3000)
                                                                        │ /data (PVC, SQLite)
                                                                        ▼
                                                  Mail: POST https://ticket-gate-tech.pkr-lab.de/v1/mail
                                                        (Bearer MAIL_WEBHOOK_TOKEN) ──► ticket-gate ──► SMTP netcup
```

| Baustein | Umsetzung |
|---|---|
| Anmeldung | Passwort (scrypt) plus 6-stelliger Mail-Code, Sperre nach 3 Fehlversuchen für 24 Stunden. Alles serverseitig in Plaudi, siehe `docs/anmeldung.md` im Chatbot-Repo |
| Mailversand | `MAIL_TRANSPORT=webhook` gegen den [Mail-Endpunkt von ticket-gate](300r0-ticket-gate.md#mail-endpunkt-für-andere-dienste). Die SMTP-Zugangsdaten liegen nur dort |
| Daten | SQLite in `/data`, PVC `plaudi-data` (2 Gi, `local-path`), ein Pod, Strategie `Recreate` |
| Image | `ghcr.io/pkr-lab/plaudi`, gebaut vom Workflow `build.yml` im Chatbot-Repo (Tag `sha-<commit>`), Pull-Secret `ghcr-pull` |
| Pod | nicht root (UID 1000), read-only Dateisystem, nur `/data` und `/tmp` beschreibbar, ohne Capabilities |

Die Mail-Code-Prüfung bleibt in Plaudi. Das Bestätigungstoken von ticket-gate lässt sich nur beim Anlegen eines Zammad-Tickets einlösen und taugt deshalb nicht als zweiter Faktor für andere Dienste.

## Konfiguration

`values.yaml` (`config:` → ConfigMap) und das SealedSecret `plaudi-secrets`. Alle Variablen des Dienstes stehen in `docs/deployment.md` des Chatbot-Repos.

| Variable | Wert | Hinweis |
|---|---|---|
| `PUBLIC_URL` | `https://plaudi-prod.pkr-lab.de` | Links in Mails werden nie aus der Anfrage gebildet |
| `COOKIE_SECURE` | `true` | Cookie mit `__Host-`-Präfix, nur über HTTPS |
| `TRUST_PROXY` | `2` | zwei Proxys davor (Traefik, cloudflared), damit die Limits die echte IP sehen |
| `MAIL_TRANSPORT` / `MAIL_WEBHOOK_URL` | `webhook` / `https://ticket-gate-tech.pkr-lab.de/v1/mail` | |
| `MAIL_FROM` | `Plaudi <info@pkr-service.de>` | wird von ticket-gate ignoriert, dort gilt `SMTP_FROM` mit `MAIL_RELAY_FROM_NAME` |
| `API_KEY` (Secret) | Anthropic-API-Schlüssel | mehrere Konten über `API_KEYS` möglich |
| `ADMIN_EMAIL` (Secret) | erster Administrator | wird bei jedem Start sichergestellt |
| `MAIL_WEBHOOK_TOKEN` (Secret) | Bearer-Token für den Mail-Endpunkt | muss dem `MAIL_RELAY_TOKEN` in ticket-gate entsprechen |

## Inbetriebnahme

1. **Image:** Im Chatbot-Repo auf `main` mergen, der Workflow `build.yml` baut `ghcr.io/pkr-lab/plaudi:sha-<commit>`. Das Package ist anfangs privat.
2. **Pull-Secret** in PROD anlegen (der Namespace entsteht beim ersten Sync, bis dahin `ImagePullBackOff` ist unkritisch, die App startet mit `replicaCount: 0`):

   ```bash
   ssh ubuntu@192.168.178.99 "sudo k3s kubectl create namespace plaudi --dry-run=client -o yaml | sudo k3s kubectl apply -f -"
   ssh ubuntu@192.168.178.99 "sudo k3s kubectl -n plaudi create secret docker-registry ghcr-pull --docker-server=ghcr.io --docker-username=<user> --docker-password=<PAT mit read:packages>"
   ```

3. **Secrets versiegeln:** `scripts/seal-plaudi.sh` fragt `API_KEY` und `ADMIN_EMAIL`, erzeugt das Relay-Token und versiegelt es zweimal: für PROD (`~/prod-sealed-secrets.pem`, Namespace `plaudi`) in `values.yaml` von Plaudi und für TECH (Namespace `ticket-gate`) als `encryptedRelayToken` in `values.yaml` von ticket-gate. Das Skript setzt `secrets.enabled: true` und `replicaCount: 1`. Der kubectl-Kontext muss TECH sein.
4. **Image-Tag** in `argocd/apps/prod/plaudi/values.yaml` unter `image.tag` eintragen.
5. PR mergen. ticket-gate muss mit dem Mail-Endpunkt laufen, bevor sich jemand anmeldet (`build-images.yml` baut das Image und bumpt `image.tag`, erst dieser Bump-PR aktiviert den Endpunkt).
6. Anmelden: *Passwort vergessen* mit der `ADMIN_EMAIL` löst eine Mail zum Festlegen des Passworts aus.

## Betrieb

- **Update:** Neuer Tag in `values.yaml`, die Daten bleiben im PVC.
- **Backup:** Alle Daten liegen in `/data/chatbot.sqlite`. Das PVC liegt auf `local-path` der prod-vm und ist **nicht** Teil eines Cluster-Backups. Sicherung: Deployment kurz auf 0 skalieren und das Verzeichnis des PVC unter `/var/lib/rancher/k3s/storage/` auf der prod-vm kopieren, danach wieder auf 1 skalieren.
- **Nur eine Instanz:** Limits, Sperren und laufende Antworten werden nicht zwischen Instanzen geteilt.
- **Notzugang:** Fällt der Mailversand aus, ist keine Anmeldung möglich. Dann kurzzeitig `MAIL_TRANSPORT: console` setzen und den Code aus dem Pod-Log lesen.
- **Kosten:** Ausgabenlimit in der Anthropic-Konsole setzen. Alle Chatinhalte gehen zur Beantwortung an die Anthropic-API.

## Rotation

| Secret | Vorgehen |
|---|---|
| `API_KEY` | In der Anthropic-Konsole neuen Schlüssel erzeugen, `scripts/seal-plaudi.sh` ausführen (erzeugt dabei auch ein neues Relay-Token), PR mergen, alten Schlüssel löschen |
| Relay-Token | `scripts/seal-plaudi.sh` ausführen, beide `values.yaml` in einem PR mergen. Zwischen dem Rollout beider Seiten schlagen Mails mit `401` fehl, die Anmeldung ist kurz nicht möglich |

## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| `ImagePullBackOff` | Pull-Secret fehlt oder Tag existiert nicht | `ghcr-pull` im Namespace `plaudi` prüfen, Tag in `values.yaml` |
| Pod startet nicht, „ADMIN_EMAIL“ fehlt | `secrets.enabled: false` | `scripts/seal-plaudi.sh` ausführen |
| Kein Code in der Mail | ticket-gate ohne Endpunkt (`404`), Token falsch (`401`), Limit (`429`) | `kubectl -n ticket-gate logs deploy/ticket-gate` (TECH), Log von Plaudi: „Mail-Webhook antwortete mit Status …“ |
| Anmeldung meldet Herkunft abgelehnt | Proxy ändert den `Host`-Header | Ingress und Tunnel prüfen, `Host` muss `plaudi-prod.pkr-lab.de` bleiben |
| Alle Nutzer teilen sich das Anmelde-Limit | `TRUST_PROXY` passt nicht zur Zahl der Proxys | Wert in `values.yaml` anpassen |
