# Rolle `argocd`

ArgoCD per Helm, Bootstrap-ApplicationSet und -Projects, Namespace-Labels, NetworkPolicies.

| | |
|---|---|
| Läuft gegen | controlplane-0 (Hub), entw-vm |
| Tags | `argocd`, `gitops` |
| Quellcode | [roles/argocd](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/argocd) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `argocd_entw_layout` | `false` | – |
| `argocd_ci_account_enabled` | `true` | – |
| `argocd_ingress_enabled` | `false` | – |
| `argocd_ui_host` | `` | – |
| `argocd_ingress_tls_crt` | `` | – |
| `argocd_ingress_tls_key` | `` | – |
| `argocd_server_nodeport_enabled` | `true` | – |
| `argocd_platform_apps` | `[alert-caller, sealed-secrets, kubeseal-webgui, lldap, monitoring, logging, ntfy, ntfy-…` | – |
| `argocd_workloads_apps` | `[alamos-apager, alamos-relay, carplay-api, github-release-watcher, isc-news-bot, mediam…` | – |
| `argocd_apply_network_policies` | `true` | – |
| `argocd_network_policy_refined_namespaces` | `[alert-caller, n8n, alamos-apager, alamos-relay, carplay-api, github-release-watcher, i…` | – |
| `argocd_network_policy_excluded_apps` | `[]` | – |
| `argocd_network_policy_extra_ingress` | `{garage: [argo-workflows], monitoring: [carplay-api, n8n, argo-workflows], n8n: [alamos-…` | – |

## Ablauf

1. argocd-namespace anlegen (idempotent)
2. argo-helm-repository hinzufügen
3. helm-repositories aktualisieren
4. helm-werte für argocd nach /tmp templaten
5. tls-secret für den ingress anlegen
6. argocd per helm installieren oder aktualisieren
7. status des helm-releases anzeigen
8. warten bis das argocd-server-deployment verfügbar ist
9. warten bis alle argocd-pods bereit sind
10. prüfen ob das initiale admin-secret existiert
11. initiales argocd-admin-passwort lesen
12. initiale argocd-zugangsdaten anzeigen (passwort verborgen)
13. initiale argocd-zugangsdaten anzeigen (passwort sichtbar)
14. bootstrap-appprojects templaten
15. bootstrap-appprojects anwenden
16. alte applicationsets platform/workloads entfernen (entw-layout)
17. bootstrap-applicationset templaten
18. bootstrap-applicationset anwenden
19. alte appprojects platform/workloads entfernen (entw-layout)
20. status der applicationsets anzeigen
21. warten bis die applicationsets verarbeitet sind
22. angelegte applications auflisten
23. applications anzeigen
24. namespaces der platform-stufe mit security-tier=platform labeln
25. namespaces der workloads-stufe mit security-tier=workload labeln
26. vorhandene namespaces lesen
27. vorhandene namespaces als fakt setzen
28. networkpolicies tier-default-ingress templaten
29. networkpolicies tier-default-ingress anwenden
30. temporäre helm-wertedatei entfernen

## Templates

- `argocd-values.yaml.j2`
- `bootstrap-applicationset.yaml.j2`
- `bootstrap-appprojects.yaml.j2`
- `bootstrap-networkpolicies.yaml.j2`

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
kubectl -n argocd get pods,applications.argoproj.io,applicationsets.argoproj.io
helm -n argocd status argocd
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Helm-Install scheitert oder wartet ewig | Pods starten nicht (Ressourcen, Image), `helm upgrade --install` Timeout (Retries in der Rolle) | `kubectl -n argocd get pods`, `describe`, Task-Ausgabe |
| Keine Applications nach der Rolle | Bootstrap-ApplicationSet nicht angewendet oder Repo-URL falsch (`argocd_repo_url`) | `kubectl -n argocd describe applicationset home-server-apps-tech`, [ArgoCD: Fehlersuche](../../capulus-core/b-kubernetes-gitops/b0010-argocd.md#fehlersuche) |
| Neue App wird abgelehnt (`not permitted in project`) | Namespace fehlt in `argocd_platform_apps` oder `argocd_workloads_apps` | Liste ergänzen, `make render-bootstrap CORE_DIR=…`, committen ([AppProjects](../../capulus-core/b-kubernetes-gitops/b0020-argocd-projects.md#fehlersuche)) |
| Initial-Passwort nicht auffindbar | Secret `argocd-initial-admin-secret` wurde gelöscht oder das Passwort geändert | [ArgoCD: Initial-Credentials](../../capulus-core/b-kubernetes-gitops/b0010-argocd.md#initial-credentials) |
| UI nicht erreichbar | Ingress oder TLS-Secret (`argocd_ingress_tls_crt`, `argocd_ingress_tls_key`), NodePort | `kubectl -n argocd get ingress,svc`, [Internes TLS](../../capulus-core/d-sicherheit/d0040-internal-tls.md#fehlersuche) |
| NetworkPolicies blockieren Verkehr | `argocd_apply_network_policies: true` mit fehlender Quelle | [Network Policies](../../capulus-core/d-sicherheit/d0030-network-policies.md#fehlersuche) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
