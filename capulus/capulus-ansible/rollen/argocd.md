# Rolle `argocd`

ArgoCD per Helm, Bootstrap-ApplicationSet und -Projects, Namespace-Labels, NetworkPolicies.

| | |
|---|---|
| Läuft gegen | homeserver (Hub), entw-vm |
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
| `argocd_network_policy_excluded_apps` | `[ollama]` | – |
| `argocd_network_policy_extra_ingress` | `{minio: [argo-workflows], monitoring: [carplay-api, n8n, argo-workflows], n8n: [alamos-…` | – |

## Ablauf

1. Create argocd namespace (idempotent)
2. Add Argo Helm repository
3. Update Helm repositories
4. Template ArgoCD Helm values to /tmp
5. Create ingress TLS secret
6. Install or upgrade ArgoCD via Helm
7. Display Helm release status
8. Wait for ArgoCD server deployment to be available
9. Wait for all ArgoCD pods to be ready
10. Check if initial admin secret exists
11. Get ArgoCD initial admin password
12. Display ArgoCD initial credentials (password redacted)
13. Display ArgoCD initial credentials (password shown)
14. Template bootstrap AppProjects
15. Apply bootstrap AppProjects
16. Remove legacy platform/workloads ApplicationSets (ENTW layout)
17. Template bootstrap ApplicationSet
18. Apply bootstrap ApplicationSet
19. Remove legacy platform/workloads AppProjects (ENTW layout)
20. Display ApplicationSet status
21. Wait for ApplicationSets to be processed
22. List created Applications
23. Display Applications
24. Label platform-tier namespaces with security-tier=platform
25. Label workloads-tier namespaces with security-tier=workload
26. Get existing namespaces
27. Set existing namespaces fact
28. Template tier-default-ingress NetworkPolicies
29. Apply tier-default-ingress NetworkPolicies
30. Remove temporary Helm values file

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
