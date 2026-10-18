# Installation

```
./install.sh
```

The script checks for the Python modules `rich` and `pyyaml` and creates a symlink at `~/.local/bin/cubi`.

## Voraussetzungen

Python 3 with `rich` and `pyyaml`. Depending on the section you need `ssh`, `kubectl`, `argocd`, `tailscale`, `gh` and `helm`.

cubi lässt sich aus dem Repo starten (`bin/cubi`) oder per Symlink im `PATH` aufrufen.
