# CoreDNS-Erweiterung — `*.homeserver` im Cluster auflösen

Pods im Cluster müssen die internen Namen unter `homeserver` (zum Beispiel `carplay-api.prod.homeserver`) auflösen können. Das Standard-CoreDNS von k3s kennt diese Zone nicht. Eine zusätzliche Konfiguration leitet sie an den DNS-Server auf dem Homeserver weiter (dnsmasq).

> **Cluster:** TECH · Ordner `argocd/apps/tech/coredns-custom/` · Objekt `ConfigMap/coredns-custom` im Namespace `kube-system` (AppProject-Namespace `coredns-custom`). `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

---

## Aufbau

Die ConfigMap `coredns-custom` enthält die Datei `homeserver.server`. k3s importiert alle `*.server`-Dateien dieser ConfigMap in die CoreDNS-Konfiguration:

```
homeserver:53 {
  forward . 192.168.178.94
  cache 30
}
```

| Zeile | Wirkung |
|---|---|
| `homeserver:53` | gilt für alle Anfragen der Zone `homeserver` (inklusive `*.tech.homeserver`, `*.prod.homeserver`, `*.dev.homeserver`) |
| `forward . 192.168.178.94` | leitet an den Homeserver (`192.168.178.94`) weiter. Dort beantwortet dnsmasq die Zone `homeserver` per Wildcard (`address=/homeserver/<ip>`, Rolle [dnsmasq](../../capulus-ansible/rollen/dnsmasq.md), siehe [DNS-Architektur](c0000-dns-architecture.md) und [Domain-Tiers](c0040-domain-tiers.md)) |
| `cache 30` | speichert Antworten 30 Sekunden |

Die Datei gilt nur für das Cluster-DNS. Hosts und Clients im Heimnetz lösen die Namen über dnsmasq (und den [Pi-hole](c0020-pihole.md)) auf, siehe [DNS-Architektur](c0000-dns-architecture.md).

---

## Fehlersuche

```bash
kubectl -n kube-system get configmap coredns-custom -o yaml
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs deploy/coredns --tail=50
# Auflösung aus einem Pod testen
kubectl run dnstest --rm -it --image=busybox:1.36 --restart=Never -- nslookup carplay-api.prod.homeserver
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Pod löst `*.homeserver` nicht auf (`NXDOMAIN`, `server can't find …`) | ConfigMap fehlt oder liegt im falschen Namespace, CoreDNS hat sie nicht geladen | `kubectl -n kube-system get configmap coredns-custom`, CoreDNS neu starten: `kubectl -n kube-system rollout restart deployment/coredns` |
| `SERVFAIL` oder Timeout bei `*.homeserver` | Weiterleitungsziel `192.168.178.94` nicht erreichbar (Homeserver aus, anderer Cluster ohne Route) | dnsmasq auf dem Homeserver prüfen (`systemctl status dnsmasq`, Rolle [dnsmasq](../../capulus-ansible/rollen/dnsmasq.md)), `nslookup <name> 192.168.178.94` vom Node |
| Falsche IP für einen Host | Eintrag im dnsmasq falsch (zum Beispiel `dnsmasq_prod_vm_hosts` für Hosts in PROD) | [DNS-Architektur](c0000-dns-architecture.md), Rolle [dnsmasq](../../capulus-ansible/rollen/dnsmasq.md) |
| Änderung wirkt erst nach Minuten | `cache 30` und Caches in Pods | kurz warten, im Pod `nslookup` erneut |
| Cluster löst nach einem Pi-hole-Update nichts mehr auf | Abhängigkeit zwischen Pi-hole und Cluster-DNS, siehe Vorfall | [Incident 50020](../5-incidents/50020-pihole-update-dns-deadlock.md) |
| Nur ein Cluster betroffen | PROD und ENTW haben ihre eigene CoreDNS-Konfiguration | Zustand je Cluster prüfen, [Domain-Tiers](c0040-domain-tiers.md) |
| ArgoCD zeigt `OutOfSync` für die ConfigMap | k3s ergänzt oder verwaltet Teile der ConfigMap | [ArgoCD](../b-kubernetes-gitops/b0010-argocd.md) |

TODO: prüfen, ob PROD und ENTW eine eigene `coredns-custom`-Konfiguration haben (im Repo steht sie nur unter `argocd/apps/tech/`).

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
