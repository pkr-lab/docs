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

## Hinweise

- Das Skript schreibt nichts in die Shell-Konfiguration. Starship wird erst aktiv, wenn `eval "$(starship init bash)"` (bzw. `zsh`, `fish`) in der Startdatei der Shell steht ([Installation](installation.md#danach)).
- Die Konfiguration ist ein Symlink auf die Datei im Checkout. Verschiebt oder löscht man den Checkout, zeigt der Link ins Leere und Starship fällt auf die Standardkonfiguration zurück. Danach `./install.sh` im neuen Checkout erneut ausführen.
- Frühere Konfigurationen liegen als `~/.config/starship.toml.bak.<Zeitstempel>` und werden nie automatisch gelöscht.
- Wer Fehler sucht: [Debugging und Fehlersuche](debugging.md).
