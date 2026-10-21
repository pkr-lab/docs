# ui-terminal-config

Ein Skript für Ubuntu Desktop, das eine einheitliche Terminal-Oberfläche einrichtet: den Prompt [Starship](https://starship.rs) samt Konfiguration, die Schrift JetBrainsMono Nerd Font und optional KI-Kommandozeilenwerkzeuge. Mehr nicht, keine Editor-Konfiguration, keine Plugins.

Repository: [pkr-lab/ui-terminal-config](https://github.com/pkr-lab/ui-terminal-config)

## Überblick

| Datei | Aufgabe |
|---|---|
| `install.sh` | installiert Starship, verlinkt die Konfiguration, installiert optional KI-CLIs und die Nerd Font; mehrfach ausführbar, jeder Schritt überspringt Vorhandenes |
| `starship-toml-template` | Starship-Konfiguration: farbige Powerline-Segmente (OS, Benutzer, Verzeichnis, Git, Sprachen, Dauer), Palette Catppuccin Mocha |

Der Ablauf von `install.sh` in drei Abschnitten:

```mermaid
flowchart LR
  A["1. Starship"] --> B["2. KI-CLIs (optional)"] --> C["3. Nerd Font"]
```

Das Skript schreibt nur an vier Stellen: `~/.local/bin` (Starship, npm-Programme), `~/.config/starship.toml` (Symlink), `~/.local/share/fonts` (Schrift) und die npm-Einstellung `prefix` (nur falls das globale Verzeichnis nicht beschreibbar ist). Die Shell-Konfiguration (`~/.bashrc`) ändert es nicht, siehe [Installation](installation.md#danach).

### Ablauf im Detail

```mermaid
flowchart TD
    S[install.sh] --> A{starship im PATH?}
    A -->|nein| A1[Download starship.rs/install.sh nach ~/.local/bin]
    A -->|ja| A2[SKIP]
    A1 --> B
    A2 --> B{Symlink ~/.config/starship.toml zeigt schon auf die Vorlage?}
    B -->|ja| B1[SKIP]
    B -->|nein| B2[Vorhandene Datei nach .bak.Zeitstempel verschieben, Symlink setzen]
    B1 --> C
    B2 --> C[npm-Präfix prüfen, ggf. auf ~/.local setzen]
    C --> D[je gewählte KI-CLI: npm install -g oder Installer]
    D --> E{JetBrainsMono Nerd Font in fc-list?}
    E -->|ja| E1[SKIP]
    E -->|nein| E2[unzip ggf. per apt, Download, entpacken, fc-cache]
```

Schritte, die schon erledigt sind, meldet das Skript als `[SKIP]`, erledigte als `[OK]`, Hinweise als `[INFO]`, Fehler als `[ERROR]` (auf stderr). Das Skript bricht bei jedem Fehler eines Befehls ab (`set -euo pipefail`), außer bei der fehlenden Node.js-Installation, die es nur meldet.

## Seiten

- [Installation](installation.md)
- [Konfiguration](konfiguration.md)
- [Nutzung](nutzung.md)
- [Debugging und Fehlersuche](debugging.md)
- [Releases](releases.md)


## Fehlersuche

Meldungen von `install.sh`, Fehlerbilder im Terminal und Checkliste: [Debugging und Fehlersuche](debugging.md).
