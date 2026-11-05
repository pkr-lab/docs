# Zwei Control-Planes im TECH-Cluster (embedded etcd)

Plan für die Umstellung des TECH-Clusters von einem k3s-Server mit SQLite auf **zwei dauerhaft laufende Control-Planes** mit embedded etcd. ControlPlane-0 ist der bisherige Homeserver, ControlPlane-1 die Hardware des bisherigen `worker-0`. Eine dritte Control-Plane (ControlPlane-2) an einem anderen Standort wird nur vorbereitet. `worker-1` bleibt ein per Wake-on-LAN gesteuerter Agent.

> **Stand (2026-11-05): in Umsetzung.** AP1 (Namensschema und Inventar) ist als Code vorbereitet, noch nicht ausgerollt. AP2 bis AP7 sind offen. Dieses Doc beschreibt den Plan und wird mit jedem Arbeitspaket ergänzt. Nach der Umsetzung wandert der Ist-Zustand in `b0000-k3s.md` und die Rollen-Docs.

Bezug: [b0000](../b-kubernetes-gitops/b0000-k3s.md) (k3s-Referenz), [20020](../../capulus-ansible/2-betrieb-hardware/20020-cluster-power-manager.md) (Power-Manager), [20030](../../capulus-ansible/2-betrieb-hardware/20030-nightly-worker-update.md) (nächtliches Update), [40090](40090-standby-cluster-ha-paar.md) (Standby-Cluster, siehe [Abgrenzung](#abgrenzung-zum-standby-cluster-40090)).

---

## Ausgangslage

| Node | IP | Rolle | Besonderheit |
|---|---|---|---|
| `homeserver` | `.94` | k3s-Server, SQLite | einziger Server, 24/7 |
| `worker-0` | `.95` | k3s-Agent | Lenovo M90q, Pentium Gold G5400T (2 Kerne / 4 Threads), 7,1 GiB RAM, kein Swap, wird heute per WoL geweckt und vom Power-Manager abgeschaltet |
| `worker-1` | `.96` | k3s-Agent | WoL-gesteuert, trägt die ENTW-VM |

Sieben Charts haben `kubernetes.io/hostname: homeserver` als `nodeSelector`, viele `local-path`-PVCs hängen per nodeAffinity am Node `homeserver`.

---

## Zielbild

| Rolle | Inventar-Host | k3s-Nodename | Gruppe | Betrieb |
|---|---|---|---|---|
| ControlPlane-0 | `controlplane-0` | `homeserver` (bleibt, siehe AP6) | `k3s_servers`, `k3s_primary` | 24/7, `cluster-init` |
| ControlPlane-1 | `controlplane-1` | `controlplane-1` (ab AP3) | `k3s_servers` | 24/7, zweiter Server |
| ControlPlane-2 | `controlplane-2` | – | eigene Gruppe, noch nicht in `k3s_servers` | später, anderer Standort, nur etcd und API |
| worker-1 | `worker-1` | `worker-1` | `k3s_agents` | WoL, unverändert |

### Grenze mit zwei etcd-Mitgliedern

Die k3s-Doku verlangt für embedded etcd eine **ungerade** Zahl Server. Mit zwei Mitgliedern liegt das Quorum bei zwei: fällt **ein** Server aus, ist die Kubernetes-API nicht mehr schreibbar. Bereits laufende Workloads laufen weiter, es gibt aber keine Änderungen, keine Neustarts von Pods mit Scheduling und kein `kubectl apply`, bis der Server zurück ist oder der Cluster aus einem Snapshot neu aufgesetzt wird. Zwei Mitglieder sind damit **nicht ausfallsicherer** als eins, die zweite Control-Plane bringt erst mit ControlPlane-2 die Ausfalltoleranz. Der Nutzen bis dahin: ein zweiter Standort der Daten (etcd-Replikat), dauerhaft laufende Kapazität für kleine Apps und die Vorbereitung auf das dritte Mitglied.

Wiederherstellung bei Verlust eines Servers: etcd-Snapshot einspielen, siehe das Runbook in AP3 (folgt) und [b0000](../b-kubernetes-gitops/b0000-k3s.md).

---

## Namensschema

Technisch (Inventar, Hostname, k3s-Nodename, Tailscale-Gerät) kleingeschrieben `controlplane-0/1/2`, weil Kubernetes und Linux keine Großbuchstaben erlauben. In Docs, Dashboard-Anzeigen und App-Texten „ControlPlane-0/1/2“. Details und Begründung: [60010](../6-hintergruende/60010-ansible-hosts-und-playbooks.md#namensschema-inventar-host-gruppen-und-k3s-nodename).

Nicht umbenannt werden die DNS-Zonen und Tiers `*.homeserver`, die Root-CA `homeserver-root-ca`, die Cloudflare-Tunnelnamen, die CRD-Gruppe `capulus.homeserver`, das Metrik-Label `clusterLabel: homeserver` und die Repo- und App-Namen (`HomeserverDashboard`).

---

## Festgelegte Entscheidungen

| Thema | Entscheidung |
|---|---|
| Schutz von ControlPlane-1 vor ungewolltem Befüllen | Taint `PreferNoSchedule`, die ausgewählten Apps bekommen Toleration und **preferred** nodeAffinity. Ein harter `NoSchedule` würde node-exporter und ServiceLB-DaemonSets ohne eigene Tolerations blockieren |
| etcd-Timeouts | Heartbeat 500 ms, Election-Timeout 5000 ms, als Variablen. Rechnung: etcd empfiehlt Heartbeat ≈ 0,5–1,5 × RTT und Election-Timeout ≥ 10 × RTT, gleiche Werte auf allen Mitgliedern. Über Tailscale mit DERP-Relay sind bis ca. 300 ms RTT realistisch. Preis: die Leaderwahl dauert bis zu 5 s länger als mit den LAN-Defaults (100/1000 ms) |
| etcd-Snapshots | alle 6 Stunden, Retention 28 pro Server (7 Tage), komprimiert, Kopie auf die NAS per NFS-Mount und Timer |
| `tls-san` | alle Server (IPs und Namen), später die Tailscale-Adresse von ControlPlane-2. Das Serving-Zertifikat muss danach rotiert werden |
| Updates | nie zwei Server gleichzeitig, vor dem Reboot Prüfung von Peer und etcd. k3s-Upgrades nacheinander, Server zuerst, `k3s_version` auf allen Servern gleich |
| Watchdogs | `resource_watchdog` löst auf Servern nur noch einen ntfy-Alarm aus, kein Poweroff. `thermal_watchdog` (Hardware-Schutz) bleibt |
| Power-Manager | ControlPlane-1 ist nicht mehr Teil von `cluster_power_manager`, `nightly_worker_wake`, `power_agent_targets`, den Power-Targets der carplay-api und der iOS-Power-Ansicht. WoL-Ziele bleiben `worker-1` und `ollama-pc` |

---

## Arbeitspakete

Die Reihenfolge ist zwingend, weil die Umstellung SQLite → etcd eine Einbahnstraße ist. AP2 muss **vor** AP3 ausgerollt sein, sonst fährt der Power-Manager ein etcd-Mitglied herunter.

| AP | Inhalt | Stand |
|---|---|---|
| 1 | Namensschema und Inventar (nur Code, keine Node-Änderung) | Code vorbereitet |
| 2 | Entkopplung vom Power-Management, Watchdog-Umstellung | ausgerollt (2026-11-05) |
| 3 | etcd und Server-Join, Runbook | offen |
| 4 | Monitoring und Alarme | offen |
| 5 | Verlagerung einzelner Apps nach ControlPlane-1 | offen, nach Messwerten |
| 6 | Umbenennung `homeserver` → `controlplane-0` | offen, Entscheidung des Nutzers nötig |
| 7 | Vorbereitung ControlPlane-2 | offen |

### AP1: Namensschema und Inventar

Umbenannt werden Inventar-Hosts, Gruppen, `host_vars`-Ordner, das Playbook (`worker-0.yml` → `controlplane-1.yml`) und die Make-Ziele (`worker-0` → `controlplane-1`, `worker-0-check` → `controlplane-1-check`). Die Node-Namen in k3s ändern sich in AP1 nicht: ControlPlane-0 setzt `k3s_node_name: homeserver`, ControlPlane-1 bleibt als Agent unter `k3s_agent_hostname: worker-0`.

Folgen, die beim Ausrollen zu beachten sind:

| Punkt | Folge |
|---|---|
| Semaphore läuft täglich 06:00 mit einem statischen Inventar vom Branch `main` | Nach dem Merge sofort `make semaphore-bootstrap`, sonst findet `site.yml` (`hosts: k3s_primary`) keine Hosts. Eine Überschreibung von `semaphore_projects` in `group_vars/all.yml` muss dieselben Inventar-Inhalte nutzen |
| `nightly_worker_wake` sucht Projekt und Template nach dem Worker-Namen | Projekt und Template für ControlPlane-1 heißen bis AP2 weiter `worker-0` / `Deploy worker-0` |
| `cluster_power_manager_workers` liest Host und MAC aus `hostvars` | der Eintrag `worker-0` liest jetzt aus `controlplane-1`, der Name bleibt bis AP2 `worker-0` (= k3s-Nodename) |
| Vault-Variablen mit alten Namen | `k3s_server_host` wird in `group_vars/k3s_servers.yml` und `group_vars/k3s_agents.yml` überschrieben (Kindgruppe schlägt `all`), eine Vault-Änderung ist nicht nötig. Aufräumen im Vault macht der Nutzer selbst |
| `ops/token-register.yaml` (capulus-core) | `stored_in` zeigt auf `host_vars/controlplane-1/vars.yml`. Der Gerätename für die Live-Prüfung des Tailscale-Keys bleibt `worker-0`, bis das Gerät in AP3 umbenannt ist |

### AP2: Entkopplung

| Änderung | Begründung |
|---|---|
| `cluster_power_manager_workers` enthält nur `worker-1`, `nightly_worker_wake` und `power_agent_targets` folgen | Der Power-Manager darf kein etcd-Mitglied herunterfahren. Die Stufenlogik bleibt generisch, mit einem Eintrag gibt es eine Stufe |
| `cluster_power_manager_target_enabled: false` auf controlplane-1 | Die Rolle entfernt Schlüssel (`cluster-power-manager@homeserver`) und sudoers-Datei. Ohne das könnte der Homeserver controlplane-1 weiter per SSH abschalten |
| `resource_watchdog_action: notify` für die Gruppe `k3s_servers` | Ein Poweroff bei 90 % Dauerlast nähme ein etcd-Mitglied mit. Der Homeserver hat bisher selbst abgeschaltet und verhält sich jetzt wie controlplane-1. `thermal_watchdog` bleibt |
| Semaphore-Projekt `worker-0` heißt `controlplane-1` | Der nächtliche Zyklus sucht Projekt und Template nach dem Worker-Namen, er kennt controlplane-1 nicht mehr |
| carplay-api: Host-ID `controlplane-1`, kein Power-Target `worker-0` | Die iOS-Power-Ansicht zeigt controlplane-1 nur als Statuskarte. Alte App-Builds zeigen bis zum Update eine falsche Karte, Wake/Shutdown für `worker-0` beantwortet der power-agent mit 400 |
| Warnung beim Herunterfahren von controlplane-0 über die App | `/poweroff-self` stoppt k3s auf controlplane-0, mit zwei Mitgliedern ist die API danach nicht schreibbar |

Beim Ausrollen gilt die Reihenfolge: erst `site.yml` (Power-Manager-Liste ohne worker-0), dann controlplane-1 (Schlüssel entfernen), sonst fährt der alte Power-Manager controlplane-1 wieder herunter.

### AP3 bis AP7

Die Inhalte stehen im jeweiligen Abschnitt, sobald das Paket umgesetzt ist. Vorab die Randbedingungen, die schon feststehen:

- **AP3:** Vor dem Join müssen die PV-Inventur (jede PV mit nodeAffinity auf `worker-0` wird verwaist, weil `controlplane-1` ein neuer Node ist) und die Plattenprüfung (etcd braucht eine SSD) erledigt sein.
- **AP5:** Die Summe der Memory-Limits aller verschobenen Apps bleibt unter ca. 2 GiB, weil ControlPlane-1 nach Abzug von System, k3s-Server und etcd nur etwa 4 GiB für Apps behält.
- **AP6:** Ein k3s-Nodename ist nicht änderbar. Optionen (Label-Override, PVs neu anlegen, Nodenamen `homeserver` behalten) werden mit Empfehlung vorgelegt, umgesetzt wird erst nach der Entscheidung des Nutzers.

---

## Abgrenzung zum Standby-Cluster (40090)

[40090](40090-standby-cluster-ha-paar.md) beschreibt einen **separaten** externen Cluster, der bei einem Ausfall Vaultwarden, n8n und ntfy samt Tunnel übernimmt (Dienst-Failover mit eigenem ArgoCD). ControlPlane-2 ist etwas anderes: ein drittes etcd-Mitglied desselben Clusters ohne Workloads, das die API bei Ausfall eines Servers erreichbar hält. Es übernimmt keine Dienste. Beide Pläne ergänzen sich, ersetzen sich aber nicht.

---

## Risiken

| Risiko | Gegenmaßnahme |
|---|---|
| Umstellung auf etcd ist nicht umkehrbar | Backup vor dem Neustart (`state.db`, Token, TLS-Verzeichnis), Rollback nur vor dem Join möglich, danach nur per etcd-Snapshot |
| API weg, sobald einer der zwei Server ausfällt | Monitoring (AP4), Snapshots, ControlPlane-2 (AP7) |
| Der Power-Manager fährt ein etcd-Mitglied herunter | AP2 zuerst ausrollen |
| `power_agent` kann ControlPlane-0 über die iOS-App abschalten (`/poweroff-self`), danach ist die API weg | Warnung in Doku und App-Dialog (AP2) |
| Weicher Taint verhindert das Befüllen von ControlPlane-1 nur als Score-Malus | Alarm auf hohe Memory-Requests auf ControlPlane-1 (AP4) |
| ControlPlane-1 hat 7,1 GiB RAM und 2 Kerne, etcd und API kosten ca. 1 bis 1,5 GiB und spürbar CPU | höhere `kube-reserved` und `system-reserved` in den `host_vars`, Obergrenze für verschobene Apps |
| `worker-1` kennt nur ControlPlane-0 als `server:` | fester Registrierungsendpunkt als spätere Verbesserung |
