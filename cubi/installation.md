# Installation

Code: [install.sh](https://github.com/pkr-lab/cubi/blob/main/install.sh), [bin/cubi](https://github.com/pkr-lab/cubi/blob/main/bin/cubi)

---

## Voraussetzungen

| Voraussetzung | Wofür | Prüfen |
|---|---|---|
| Python 3 (der Code nutzt `X \| None`-Typen, also Python 3.10 oder neuer) | Laufzeit | `python3 --version` |
| Python-Module `rich` und `pyyaml` | Ausgabe, Konfiguration | `python3 -c "import rich, yaml"` |
| Linux oder macOS mit TTY | `termios` für die Tastatureingabe | Windows wird nicht unterstützt |
| `ssh` | Bereiche Connect und Tailscale | `ssh -V` |
| `kubectl` | ArgoCD, Auto-Sync, Feature-Testing, Skalieren, Kubernetes | `kubectl version --client` |
| `argocd` | Diff, Sync, Refresh, Historie, Login | `argocd version --client` |
| `gh` | Bereich GitHub, Login, Branch-Prüfung bei Feature-Testing | `gh --version` |
| `tailscale` | Bereich Tailscale und Login | `tailscale version` |
| `helm` | Bereich Helm | `helm version` |

Die Werkzeuge werden erst beim Aufruf gebraucht. Fehlt eines, meldet cubi nur beim Öffnen des betroffenen Bereichs `<werkzeug> ist nicht installiert oder nicht im PATH`. Die anderen Bereiche laufen weiter.

---

## Installieren

```bash
git clone https://github.com/pkr-lab/cubi.git
cd cubi
./install.sh
```

Das Skript:

1. prüft mit Python, ob `rich` und `yaml` importierbar sind, und bricht sonst mit `Fehlende Python-Module: …` und Exit-Code 1 ab,
2. macht `bin/cubi` ausführbar,
3. legt `~/.local/bin/` an und setzt dort den Symlink `cubi` auf `bin/cubi`,
4. warnt auf stderr, wenn `~/.local/bin` nicht im `PATH` ist.

Fehlende Module installiert man über die Paketverwaltung der Distribution oder in einer virtuellen Umgebung:

```bash
sudo apt install python3-rich python3-yaml     # Debian/Ubuntu
# oder
python3 -m pip install --user rich pyyaml
```

cubi lässt sich auch ohne Installation direkt aus dem Checkout starten: `./bin/cubi`. `bin/cubi` ergänzt `src/` selbst im Suchpfad, `PYTHONPATH` muss nicht gesetzt werden.

---

## Prüfen

```bash
cubi --help        # listet die Bereiche, braucht kein TTY
cubi login         # zeigt, welche Werkzeuge und Logins fehlen
```

`cubi` ohne Argument meldet in Pipes oder Skripten `cubi benötigt ein interaktives Terminal` (Exit-Code 1). Nur `--help` funktioniert ohne TTY.

---

## Aktualisieren und Entfernen

- Aktualisieren: `git pull` im Checkout. Der Symlink zeigt weiter auf `bin/cubi`.
- Entfernen: `rm ~/.local/bin/cubi` und optional `~/.config/cubi/`. cubi legt sonst nichts auf dem Rechner ab. Im Cluster entsteht nur die ConfigMap `cubi-state` im ArgoCD-Namespace, siehe [ArgoCD-Features](argocd-features.md).

Releases entstehen aus den Commits, siehe [Releases](releases.md).

---

## Fehlerbilder bei der Installation

| Meldung | Ursache | Lösung |
|---|---|---|
| `Fehlende Python-Module: rich, yaml` | Module für das `python3` des Skripts fehlen | Module installieren (siehe oben), mit demselben `python3` prüfen, das auch `bin/cubi` startet (`#!/usr/bin/env python3`) |
| `~/.local/bin ist nicht im PATH` | Verzeichnis fehlt im `PATH` | `export PATH="$HOME/.local/bin:$PATH"` in `~/.bashrc` oder `~/.zshrc` |
| `cubi: command not found` nach der Installation | neue Shell nötig oder `PATH` fehlt | Shell neu starten, `ls -l ~/.local/bin/cubi` prüfen |
| `ModuleNotFoundError: No module named 'rich'` beim Start | anderes `python3` als bei der Installation (zum Beispiel aktive virtuelle Umgebung) | Umgebung deaktivieren oder dort `pip install rich pyyaml` |
| `SyntaxError` bei `X \| None` | Python älter als 3.10 | Python aktualisieren |
| Symlink zeigt ins Leere | Checkout verschoben oder gelöscht | `./install.sh` im neuen Checkout erneut ausführen |

Weitere Fehler siehe [Debugging und Fehlersuche](debugging.md).
