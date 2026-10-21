# Installation

## Voraussetzungen

| Was | Wofür |
|---|---|
| Ubuntu Desktop | Zielsystem |
| `curl` | wird vom Skript für alle Downloads gebraucht (`sudo apt install curl`) |
| `nodejs`, `npm` | nur für die KI-CLIs `claude`, `codex`, `gemini` (`sudo apt install nodejs npm`) |
| `unzip` | wird bei Bedarf automatisch per `sudo apt-get install unzip` installiert |

## Setup

```sh
git clone https://github.com/pkr-lab/ui-terminal-config.git ~/ui-terminal-config
cd ~/ui-terminal-config
./install.sh
```

Das Skript kann beliebig oft laufen. Es überspringt, was schon installiert ist.

## Danach

- `~/.local/bin` in den `PATH` aufnehmen, damit `starship` gefunden wird
- Starship in der Shell aktivieren, z. B. in `~/.bashrc`: `eval "$(starship init bash)"`
- In der Terminal-Anwendung die Schrift `JetBrainsMono Nerd Font Mono` einstellen

## Probleme bei der Installation

| Symptom | Ursache | Lösung |
|---|---|---|
| `curl: command not found` | `curl` fehlt | `sudo apt install curl` |
| `[ERROR] claude -- npm not found, install Node.js first` | `npm` fehlt (Standard installiert Claude Code) | `sudo apt install nodejs npm` oder `INSTALL_CLAUDE=false ./install.sh` |
| `starship: command not found` nach dem Lauf | `~/.local/bin` nicht im `PATH` | `PATH` ergänzen (siehe oben) |
| Prompt zeigt Kästchen | Terminal nutzt nicht die Nerd Font | Schrift einstellen (siehe oben) |

Alle Meldungen des Skripts und weitere Fehlerbilder: [Debugging und Fehlersuche](debugging.md).
