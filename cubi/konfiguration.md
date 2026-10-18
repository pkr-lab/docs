# Konfiguration

Hosts, ArgoCD contexts, GitHub repos and the capulus-core checkout (`tokens.repo`, default `../capulus-core` next to cubi) live in `config/targets.yaml`. The file is looked up in this order:

1. The file named by the `CUBI_CONFIG` environment variable
2. `~/.config/cubi/targets.yaml`
3. `config/targets.yaml` in the repo

The GitHub Kanban board uses the project's "Status" field (or the first single-select field with columns). Pin a default board under `github.board`, otherwise cubi lets you pick one from `gh project list` for the repo's owner:

```yaml
github:
  board:
    owner: pkr-lab
    number: 3
```

Moving cards and editing issue labels/assignees needs a GitHub token with the `project` scope, which `gh auth refresh -s project,read:project` (also reachable from `cubi login` → GitHub → "Projekt-Scope aktivieren") adds on top of the default login.

## Aufbau von `targets.yaml`

| Schlüssel | Bedeutung |
|---|---|
| `connect.vm[]`, `connect.hosts[]` | Einträge mit `name` und `ssh` (SSH-Alias aus `~/.ssh/config`) |
| `argocd.namespace` | Namespace der ArgoCD-Installation (`argocd`) |
| `argocd.contexts[]` | ArgoCD-Zugänge: `core: true` nutzt das Kubernetes-Backend, sonst `server`, `plaintext`, `insecure`, `username` |
| `github.default_repo`, `github.repos[]` | Repos für den GitHub-Abschnitt |
| `github.board.owner`, `github.board.number` | optionales Standard-Kanban-Board |
| `tokens.repo` | Checkout von capulus-core mit dem Token-Register |
| `tokens.docs_repo` | Checkout von pkr-lab-docs für Runbooks mit dem Präfix `pkr-lab-docs:` (Standard: `../pkr-lab-docs` neben cubi) |
