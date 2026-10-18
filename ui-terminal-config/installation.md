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
