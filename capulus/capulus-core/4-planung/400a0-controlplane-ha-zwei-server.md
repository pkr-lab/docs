# Zwei Control-Planes im TECH-Cluster (embedded etcd)

Plan für die Umstellung des TECH-Clusters von einem k3s-Server mit SQLite auf **zwei dauerhaft laufende Control-Planes** mit embedded etcd. ControlPlane-0 ist der bisherige Homeserver, ControlPlane-1 die Hardware des bisherigen `worker-0`. Eine dritte Control-Plane (ControlPlane-2) an einem anderen Standort wird nur vorbereitet. `worker-1` bleibt ein per Wake-on-LAN gesteuerter Agent.

> **Stand (2026-11-05): in Umsetzung.** AP1 bis AP3 sind ausgerollt: controlplane-0 und controlplane-1 bilden einen etcd-Cluster mit zwei Mitgliedern. AP4 ist ausgerollt, AP5 läuft in Wellen, AP6 und AP7 sind offen. Dieses Doc beschreibt den Plan und wird mit jedem Arbeitspaket ergänzt. Nach der Umsetzung wandert der Ist-Zustand in `b0000-k3s.md` und die Rollen-Docs.

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
| 1 | Namensschema und Inventar (nur Code, keine Node-Änderung) | ausgerollt |
| 2 | Entkopplung vom Power-Management, Watchdog-Umstellung | ausgerollt (2026-11-05) |
| 3 | etcd und Server-Join, Runbook | durchgeführt (2026-11-05) |
| 4 | Monitoring und Alarme | ausgerollt |
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

### AP3: etcd und Server-Join

Die Rolle `k3s` bekommt die Modi `init` und `join` ([k3s](../../capulus-ansible/rollen/k3s.md)). controlplane-0 wird `init` (SQLite → embedded etcd), controlplane-1 wird `join`. `k3s_version` ist in `group_vars/k3s_servers.yml` und `group_vars/k3s_agents.yml` auf die laufende Version festgeschrieben, nicht in `all.yml`: Dort gilt sie auch für `prod-vm` und `entw-vm`, deren Version unbekannt ist, ein Pin würde sie herunterstufen.

| Änderung | Begründung |
|---|---|
| etcd-Timeouts 500/5000 ms schon jetzt | Alle Mitglieder müssen gleich konfiguriert sein. So braucht der spätere Beitritt von ControlPlane-2 keinen Neustart der anderen Server |
| Snapshots alle 6 h, Retention 28 pro Server, Kopie auf die NAS per NFS-Mount und Timer | Zwei Mitglieder tragen kein Ausfallrisiko des Datastores: Die Snapshots sind der Weg zurück. Die Kopie liegt außerhalb der Hosts |
| `tls-san` aus allen Servern (IP, Inventarname, Nodename) plus `k3s_tls_san_extra` | Clients sollen jeden Server ansprechen können, später auch die Tailscale-Adresse von ControlPlane-2 |
| UFW: 6443 nur aus LAN/Tailnet/Trusted Hosts, 2379–2380 nur von den anderen Servern | Die offene 6443-Regel auf controlplane-1 wird erst nach dem Anlegen der Regeln mit Quelladresse gelöscht, niemand sperrt sich aus |
| Taint `PreferNoSchedule` und höhere `kube-reserved` (1,5 Gi) auf controlplane-1 | Weicher Taint: ungepinnte Pods meiden den Node, DaemonSets laufen weiter. etcd und API kosten dort 1 bis 1,5 GiB |
| Rolle `k3s_reboot_guard`, `k3s-servers-update.yml`, `k3s-upgrade.yml` | Nie zwei Server gleichzeitig neu starten oder aktualisieren |

**Wichtig vor dem Merge:** Der tägliche Semaphore-Lauf (06:00) schreibt nach dem Merge `cluster-init` in die Config von controlplane-0. Ein ungeplanter Neustart von k3s (oder des Hosts) würde dann die Konvertierung auslösen. Den PR deshalb erst am Beginn des Wartungsfensters mergen. Der PR für `ops/token-register.yaml` (`live_check_ref: controlplane-1`) erst nach dem Umbenennen des Tailscale-Geräts, sonst schlägt die Ablaufprüfung fehl.

**Mit zwei Mitgliedern ist die API bei jedem Neustart eines Servers kurz weg**, nicht nur bei Ausfällen. Die Pods laufen weiter, `kubectl` antwortet erst wieder, wenn beide Server laufen.

#### Voraussetzungen (vor dem Wartungsfenster)

1. **PV-Inventur.** Der Node `worker-0` verschwindet, jede gebundene `local-path`-PV mit Affinität auf `worker-0` wäre danach verwaist. Auf controlplane-0:
   ```bash
   kubectl get pv -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NS:.spec.claimRef.namespace,CLAIM:.spec.claimRef.name,SC:.spec.storageClassName,NODE:.spec.nodeAffinity.required.nodeSelectorTerms[0].matchExpressions[0].values[0] | awk 'NR==1 || $6=="worker-0"'
   kubectl get pods -A -o wide --field-selector spec.nodeName=worker-0
   ```
   In der Spalte `STATUS` darf kein `Bound` stehen, und auf worker-0 darf kein Pod mit PVC laufen. Eine `Bound`-PV auf worker-0 stoppt das Vorhaben: erst die Daten umziehen. Auf worker-0 war `/var/lib/rancher/k3s/storage` leer.
2. **Platte.** controlplane-1 hat eine NVMe-SSD (`ROTA=0`), 46 GB frei.
3. **NAS-Verzeichnis für die Snapshots.** Die Kopie nutzt den bestehenden Export `/volume1/k8s-storage` der NAS (`192.168.178.97`, die Server `.94`, `.95` und `.96` sind dort bereits zugelassen) mit dem Unterverzeichnis `k3s-etcd-snapshots`. Anlegen und testen, auf controlplane-0:
   ```bash
   sudo mkdir -p /mnt/nas-test && sudo mount -t nfs 192.168.178.97:/volume1/k8s-storage /mnt/nas-test
   sudo mkdir -p /mnt/nas-test/k3s-etcd-snapshots && sudo touch /mnt/nas-test/k3s-etcd-snapshots/.test && ls -la /mnt/nas-test/k3s-etcd-snapshots
   sudo rm /mnt/nas-test/k3s-etcd-snapshots/.test; sudo umount /mnt/nas-test
   ```
   Die Werte stehen in `group_vars/k3s_servers.yml` (`k3s_snapshot_copy_enabled`, `k3s_snapshot_nfs_server`, `k3s_snapshot_nfs_export`). Der Rsync-Aufruf kopiert ohne Eigentümer und Rechte, ein Export mit `root_squash` genügt.

#### Ablauf

1. **Fenster ankündigen.** ntfy-Nachricht an `Home-Lab-System`. In Alertmanager (Grafana → Alerting → Silences) eine Silence für `alertname=worker-0Down` setzen, sonst ruft der Alarm-Anrufer an, sobald worker-0 abgeschaltet wird.
2. **Backup auf controlplane-0** (k3s stoppen, damit `state.db` konsistent ist; die Pods laufen weiter):
   ```bash
   sudo systemctl stop k3s
   sudo tar -czf /root/k3s-backup-$(date +%Y%m%d-%H%M).tar.gz -C / etc/rancher/k3s var/lib/rancher/k3s/server/db var/lib/rancher/k3s/server/token var/lib/rancher/k3s/server/node-token var/lib/rancher/k3s/server/tls var/lib/rancher/k3s/server/cred
   sudo systemctl start k3s
   sudo sh -c 'ls -lh /root/k3s-backup-*.tar.gz'
   ```
   Das Archiv danach vom Host wegkopieren (Workstation oder NAS).
3. **PR mergen, Config schreiben.** Dry-Run, dann ausrollen. Die Rolle schreibt nur die Config (`cluster-init`, etcd-Argumente, Snapshots, `tls-san`) und startet k3s nicht neu:
   ```bash
   ansible-playbook -i inventory/hosts.yml site.yml --tags k3s --check --diff --ask-vault-pass
   make k3s
   ```
   Im Diff nur `/etc/rancher/k3s/config.yaml` und die Firewall-Regeln für 2379–2380. Kein Installer-Lauf, denn die Version stimmt bereits.
4. **Bewusster Neustart (Konvertierung)** auf controlplane-0. Die API ist ein bis drei Minuten weg:
   ```bash
   sudo systemctl restart k3s
   sudo journalctl -u k3s -n 80 --no-pager | grep -i -E "etcd|cluster-init|migrat"
   kubectl get nodes
   sudo ls /var/lib/rancher/k3s/server/db/
   kubectl get --raw=/readyz/etcd
   kubectl get nodes -l node-role.kubernetes.io/etcd
   curl -s http://127.0.0.1:2381/metrics | grep -E "^etcd_server_(has_leader|is_leader)"
   sudo k3s etcd-snapshot save --name vor-join
   sudo k3s etcd-snapshot ls
   ```
   Erwartet: Verzeichnis `etcd/` neben `state.db`, der Node hat die Rollen `control-plane,etcd` (ein Label `master` setzt diese k3s-Version nicht mehr), `etcd_server_has_leader 1`, ein Snapshot `vor-join` wird gelistet. Erst weitermachen, wenn das stimmt.
5. **worker-0 aus dem Cluster nehmen.** Auf controlplane-0:
   ```bash
   kubectl drain worker-0 --ignore-daemonsets --delete-emptydir-data --timeout=180s
   ```
   Auf worker-0:
   ```bash
   sudo /usr/local/bin/k3s-agent-uninstall.sh
   ```
   Wieder auf controlplane-0:
   ```bash
   kubectl delete node worker-0
   ```
6. **controlplane-1 als Server beitreten lassen** (Workstation). Beim ersten Lauf kann der Dry-Run spät scheitern, weil k3s noch nicht installiert ist. Die Rolle bricht ab, falls noch `k3s-agent.service` existiert:
   ```bash
   ansible-playbook -i inventory/hosts.yml controlplane-1.yml --check --diff --ask-vault-pass
   make controlplane-1
   ```
   Die Rolle setzt den OS-Hostnamen auf `controlplane-1`, installiert k3s als Server in derselben Version, wartet auf `Ready`, `/readyz/etcd` und zwei etcd-Mitglieder. Danach hebt der Uncordon-Play ein evtl. gesetztes Cordon auf. Der Tailscale-Name des Geräts wird dabei ebenfalls auf `controlplane-1` umbenannt.
7. **Verifikation** auf controlplane-0:
   ```bash
   kubectl get nodes -o wide
   kubectl get nodes -l node-role.kubernetes.io/etcd
   kubectl describe node controlplane-1 | grep -A2 Taints
   kubectl -n kube-system get pods -o wide
   curl -s http://127.0.0.1:2381/metrics | grep -E "^etcd_server_(has_leader|is_leader)"
   sudo k3s etcd-snapshot ls
   ```
   Erwartet: zwei Nodes (`homeserver` und `controlplane-1`) mit `control-plane,etcd` (ein Label `master` setzt diese k3s-Version nicht mehr), beide `Ready`, Taint `capulus.homeserver/protected=true:PreferNoSchedule` auf controlplane-1. Der k3s-Nodename von controlplane-0 bleibt `homeserver` (Entscheidung in AP6).
8. **Neustarttest.** `ssh ubuntu@controlplane-1 sudo reboot`. Während des Neustarts ist die API weg (zwei Mitglieder, ein Quorum von zwei), Pods laufen weiter. Prüfen über einen Dienst (zum Beispiel die Vaultwarden- oder ArgoCD-Oberfläche) und danach `kubectl get nodes`.
9. **Zertifikat von controlplane-0 rotieren**, damit die neuen Namen und die IP von controlplane-1 im Serving-Zertifikat stehen. Nicht blockierend für den Beitritt. Das Verfahren stammt aus Community-Quellen, nicht aus der offiziellen Doku. Bei der Umstellung am 2026-11-05 hat es auf k3s v1.36.5 funktioniert: Das neue Zertifikat enthielt danach `controlplane-0`, `controlplane-1`, `homeserver` und beide IPs:
   ```bash
   sudo cp -a /var/lib/rancher/k3s/server/tls /root/k3s-tls-backup-$(date +%Y%m%d)
   kubectl -n kube-system delete secret k3s-serving
   sudo mv /var/lib/rancher/k3s/server/tls/dynamic-cert.json /root/dynamic-cert.json.bak
   sudo systemctl restart k3s
   echo | openssl s_client -connect 192.168.178.94:6443 2>/dev/null | openssl x509 -noout -ext subjectAltName
   ```
   Erwartet: `192.168.178.95`, `controlplane-0`, `controlplane-1` stehen in der Liste. (Ablauf bei der Umstellung: Secret löschen, `dynamic-cert.json` wegschieben, Neustart, ein bis zwei Minuten ohne API.) Erscheint nichts Neues, ist das Zertifikat nicht neu erzeugt worden: das TLS-Backup zurückspielen und den Fund melden, nicht wiederholen.
10. **Aufräumen.** Alertmanager-Silence entfernen, ntfy-Nachricht „Wartung beendet“, PR für `ops/token-register.yaml` mergen.

#### Rollback

| Zeitpunkt | Vorgehen |
|---|---|
| Vor Schritt 4 (Neustart) | Das Backup ist nicht nötig. `cluster-init` aus der Config entfernen (Rolle mit `k3s_mode: single`) und k3s weiterlaufen lassen |
| Nach Schritt 4, vor dem Beitritt | `sudo systemctl stop k3s`, `sudo mv /var/lib/rancher/k3s/server/db/etcd /root/etcd-verworfen`, `cluster-init` und die etcd-Schlüssel aus der Config nehmen, die Dateien aus dem Backup zurückspielen (`state.db`), k3s starten. Alle Änderungen seit dem Backup gehen verloren |
| Nach dem Beitritt | Kein Weg zurück zu SQLite. Wiederherstellung aus einem etcd-Snapshot: k3s auf **beiden** Servern stoppen, auf controlplane-0 `sudo k3s server --cluster-reset --cluster-reset-restore-path=<Snapshot>`, k3s starten; auf controlplane-1 `sudo rm -rf /var/lib/rancher/k3s/server/db/` und starten |
| Verlust eines Servers | Der verbleibende Server hat kein Quorum. Auf ihm `sudo systemctl stop k3s`, `sudo k3s server --cluster-reset`, starten. Der neue oder reparierte zweite Server tritt wie in Schritt 6 neu bei (vorher `kubectl delete node <name>`) |

#### Updates und Upgrades danach

- `make k3s-servers-update-check` und `make k3s-servers-update` aktualisieren die Server nacheinander (apt, Neustart nur nach etcd-Prüfung). Nur von der Workstation, nicht in Semaphore.
- `make k3s-upgrade-check` und `make k3s-upgrade` heben k3s auf `k3s_version`: erst controlplane-0, dann controlplane-1, dann die Agents. Die Version vorher in beiden `group_vars`-Dateien ändern.

### AP4: Monitoring und Alarme

Begründungen der Regeln: [60060](../6-hintergruende/60060-monitoring-und-alerting.md#regeln-vmrule).

| Änderung | Begründung |
|---|---|
| `vmrule-worker-0` → `vmrule-controlplane-1` (`ControlPlane1Down`) ohne Cordon-Ausnahme | Der Node läuft dauerhaft, jede Abwesenheit ist ein Befund. `absent(…== 1)` fängt auch einen entfernten Node |
| `vmrule-etcd` (kein Leader, Mitglieder fehlen, Peer verloren, Leaderwechsel, WAL-fsync, Datenbankgröße) | Mit zwei Mitgliedern ist die API bei jedem Ausfall weg, die Snapshots sind der einzige Weg zurück |
| Snapshot-Kopie meldet Fehler selbst per ntfy, keine Regel auf `node_systemd_*` | Das node-exporter-DaemonSet hat den systemd-Collector nicht, eine solche Regel würde nie feuern. Ausbleibende Läufe fallen so nicht auf; wer das braucht, aktiviert den Collector im DaemonSet |
| `vmstaticscrape-etcd` auf `:2381` beider Server | Die Metriken kommen aus `etcd-expose-metrics`. Sollwert `etcd.expectedMembers` steigt mit ControlPlane-2 auf 3 |
| controlplane-1 im `NodeDownPaging` | Sein Ausfall nimmt die Kubernetes-API, Telefon-Alarm nach 5 Minuten (zusätzlich zu `KubeApiDownPaging`) |
| `ControlPlane1MemoryRequestsHigh` | Der weiche Taint hält ungepinnte Pods nicht sicher fern |
| smartctl-Scrape für controlplane-1 | Der Node läuft jetzt dauerhaft |
| Sprachnamen im alert-caller: „Control Plane null/eins“ | Der Telefon-Alarm nannte `192.168.178.95` „Worker null“. Die Quelle `caller.py` gilt auch für die Alert-Caller-VM: Die Rolle `alert_caller` pinnt Tag und SHA-256 in `alert_caller_core_ref`, nach dem Release ist dort ein Bump nötig |

### AP5: Apps nach controlplane-1

Muster und Begründung: [60040](../6-hintergruende/60040-helm-charts-tech.md#scheduling-auf-controlplane-1-toleration-und-bevorzugte-affinity).

**Messung vom 2026-11-05** (vor Welle 1): controlplane-1 hat 3638 MiB zuweisbaren Speicher (7292 MiB abzüglich 1 GiB `system-reserved`, 1,5 GiB `kube-reserved` und 15 % Eviction-Schwelle). Das Working Set lag bei 2518 MiB, davon ca. 1,2 GiB `k3s-server`. `MemAvailable` lag bei 5172 MB. Auf dem Node liefen nur fünf DaemonSet-Pods (Requests 128 Mi, Limits 256 Mi). Der tatsächliche Verbrauch der Kandidaten ist klein: ticket-gate 6 Mi, pacman 11 Mi, alamos-relay und alamos-apager je 14 Mi, ntfy-bridge 16 Mi, cloudflared 17 Mi je Pod, headlamp 17 Mi, CoreDNS 41 Mi, kubeseal-webgui 104 Mi, uptime-kuma 179 Mi, zusammen etwa 0,5 GiB gegenüber Limits von etwa 1,7 GiB.

**Fund auf dem Host:** Auf controlplane-1 läuft außerhalb von Kubernetes ein Docker mit sieben Containern: `tinyteller-backend`/`-frontend`, `day-pilot-frontend`/`-backend`/`-db`/`-cache` und `node-exporter-cadvisor`. Mehrere melden `unhealthy`. Sie stehen in keinem Repo und kosten RAM (dockerd, cadvisor, Backends, Postgres, Redis). Seit controlplane-1 24/7 läuft, laufen sie ebenfalls dauerhaft. Docker setzt eigene iptables-Regeln neben denen von k3s und Flannel.

**CoreDNS:** Das Deployment gehört dem k3s-Addon-Controller (`objectset.rio.cattle.io/owner-name: coredns`, Manifest `/var/lib/rancher/k3s/server/manifests/coredns.yaml`, `replicas: 1`). Eine manuelle Skalierung kann beim Neuanwenden des Addons zurückgesetzt werden. Das Deployment hat bereits `topologySpreadConstraints` auf `kubernetes.io/hostname` (`DoNotSchedule`), ein zweites Replikat läge deshalb automatisch auf dem anderen Server.

| Welle | Apps | Stand |
|---|---|---|
| 1 | ticket-gate, github-release-watcher, pacman, ntfy-bridge, alamos-apager, alamos-relay | Code vorbereitet |
| 2 | cloudflared (verteilt auf beide Server), headlamp, kubeseal-webgui (bevorzugt controlplane-1) | Code vorbereitet |
| 3 | CoreDNS (zweites Replikat) | offen, siehe oben |
| 4 | uptime-kuma | offen, braucht Datenmigration |

### AP6 und AP7

Die Inhalte stehen im jeweiligen Abschnitt, sobald das Paket umgesetzt ist. Vorab die Randbedingungen, die schon feststehen:

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
