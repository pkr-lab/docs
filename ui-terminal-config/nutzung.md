# Nutzung

```sh
./install.sh
INSTALL_CLAUDE=false ./install.sh
INSTALL_CODEX=true INSTALL_GEMINI=true ./install.sh
INSTALL_AGY=true ./install.sh
```

| Aufgabe | Vorgehen |
|---|---|
| Alles Standardmäßige einrichten | `./install.sh` (Starship, Konfiguration, Nerd Font, Claude Code) |
| Claude Code weglassen | `INSTALL_CLAUDE=false ./install.sh` |
| Weitere KI-CLIs ergänzen | die jeweilige Variable auf `true` setzen; mehrere gleichzeitig möglich |
| Palette wechseln | `palette` in `starship-toml-template` ändern, die neue Shell lädt sie automatisch |
| Aktualisieren | `git pull`, dann `./install.sh` erneut ausführen |
