# Rolle `k3s_reboot_guard`

Gibt Neustarts und Upgrades der k3s-Server nur frei, wenn der andere Server und etcd gesund sind.

| | |
|---|---|
| Läuft gegen | controlplane-0, controlplane-1 (Hosts außerhalb von `k3s_servers` bekommen immer die Freigabe) |
| Tags | – |
| Quellcode | [roles/k3s_reboot_guard](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/k3s_reboot_guard) |

Mit zwei etcd-Mitgliedern nimmt ein Neustart des einen Servers, während der andere ausfällt oder noch hochfährt, die Kubernetes-API. Die Rolle prüft deshalb vorher. Sie wird von `worker_apt_update`, `common` (automatischer Reboot) und `k3s-upgrade.yml` eingebunden und setzt den Fakt `k3s_reboot_guard_ok`.

## Variablen

| Variable | Default | Bedeutung |
|---|---|---|
| `k3s_reboot_guard_peer_min_uptime_seconds` | `300` | so lange muss der Peer mindestens laufen, damit sein etcd-Mitglied wieder synchron ist |
| `k3s_reboot_guard_kubeconfig` | `/etc/rancher/k3s/k3s.yaml` | kubeconfig auf dem Peer |

## Ablauf

1. Peer-Server bestimmen (alle anderen Hosts der Gruppe `k3s_servers`), abbrechen wenn keiner existiert
2. Peer-Node auf `Ready` prüfen (per kubectl auf dem Peer)
3. `/readyz/etcd` über die API des Peers prüfen
4. Laufzeit des Peers lesen
5. `k3s_reboot_guard_ok: true` setzen, bei jedem Fehler `false` plus Warnung (die Aufrufer überspringen dann den Neustart, ein Fehler im Playbook entsteht nicht)

## Fehlersuche

| Symptom | Ursache | Lösung |
|---|---|---|
| „Neustart von … nicht freigegeben“ | Peer nicht `Ready`, etcd nicht ok oder Peer erst kurz online | `kubectl get nodes`, `kubectl get --raw=/readyz/etcd` auf dem Peer, später erneut ausführen |
| `k3s-upgrade.yml` bricht ab | derselbe Grund, dort ist die Freigabe Pflicht | Peer zuerst reparieren |

[Alle Rollen](index.md)
