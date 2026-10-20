# Nutzung

## Bedienung

- Up and down arrows move, Enter confirms, Esc goes back one level or cancels, Ctrl+C cancels and exits.
- In multi-selections, Space toggles an entry and `a` toggles all.
- In tables, `r` reloads. The footer shows the extra keys of the current section.

Alle Tasten, das Inline-Verhalten und die Stellen mit Mehrfachauswahl stehen unter [Bedienung](bedienung.md).

Abschnitte direkt öffnen:

```bash
cubi
cubi argocd
cubi autosync
cubi branch
cubi tokens
cubi login
```

## Tokens

`cubi tokens` reads `ops/token-register.yaml` from capulus-core and lists every token with its age and expiry date, most urgent first. Enter opens the token's menu:

- **Rotieren** (rotate): for `scripted` tokens cubi runs the script named in the register (`rotation.command`) inside the repo after a confirmation. For `manual` tokens it shows the preparation hint, where the token is stored and the overlap rules, and opens the runbook on request. `auto` tokens need nothing.
- **Als rotiert im Register eintragen** (record rotation): sets `issued` (and optionally `expires`) in the register without reformatting the rest of the file. After a successful script run cubi offers this step automatically.
- **Details** and **Runbook lesen** (read runbook).

`c` runs `scripts/check-token-register.py`. cubi never commits: the register and any re-sealed SealedSecrets stay as working-tree changes, one PR per rotation.

## Hinweise

- `cubi` never handles passwords or tokens itself. Logins run through `argocd`, `gh` and `tailscale`, token rotation through the scripts of capulus-core. The register holds no values, only dates.
- The ArgoCD status is read straight from the cluster with `kubectl`, so it works even when the ArgoCD server is unreachable.
