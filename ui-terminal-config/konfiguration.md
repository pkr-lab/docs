# Konfiguration

## Umgebungsvariablen von `install.sh`

| Variable | Standard | Wirkung |
|---|---|---|
| `INSTALL_CLAUDE` | `true` | installiert Claude Code (`@anthropic-ai/claude-code`) per `npm install -g` |
| `INSTALL_CODEX` | `false` | installiert die Codex CLI (`@openai/codex`) per npm |
| `INSTALL_GEMINI` | `false` | installiert die Gemini CLI (`@google/gemini-cli`) per npm |
| `INSTALL_AGY` | `false` | installiert die Antigravity CLI über deren eigenen Installer |

Ist das globale npm-Verzeichnis nicht beschreibbar, setzt das Skript den npm-Präfix auf `~/.local`.

## Starship (`starship-toml-template`)

Das Skript legt einen Symlink `~/.config/starship.toml` auf die Vorlage im Repo an. Eine vorhandene, abweichende Datei wird vorher nach `~/.config/starship.toml.bak.<Zeitstempel>` verschoben. Änderungen an der Vorlage wirken damit sofort.

| Abschnitt | Inhalt |
|---|---|
| `format` | Aufbau der Prompt-Zeile aus Powerline-Segmenten |
| `[os]`, `[username]`, `[directory]` | Symbole, Farben und Pfadkürzungen |
| `[git_branch]`, `[git_status]` | Git-Anzeige |
| `[nodejs]`, `[c]`, `[rust]`, `[golang]`, `[php]`, `[java]`, `[kotlin]`, `[haskell]`, `[python]`, `[conda]` | Sprachsegmente |
| `[docker_context]`, `[time]`, `[cmd_duration]`, `[line_break]`, `[character]` | weitere Segmente |
| `palette` | aktive Farbpalette, gesetzt auf `catppuccin_mocha`; die Vorlage enthält außerdem `catppuccin_frappe`, `catppuccin_latte` und `catppuccin_macchiato` |

Die Palette wechselt man mit der Zeile `palette = '…'`.

### Weitere Einstellungen der Vorlage

| Schlüssel | Wert | Wirkung |
|---|---|---|
| `command_timeout` | `5000` ms | Obergrenze für externe Befehle, die ein Segment aufruft (zum Beispiel Git). Bei Überschreitung fehlt das Segment |
| `scan_timeout` | `100` ms | Obergrenze für das Durchsuchen des Verzeichnisses nach Projektdateien |
| `[directory] truncation_length` | `3` | zeigt höchstens drei Pfadebenen, davor `…/` |
| `[cmd_duration]` | `show_milliseconds = true`, `min_time_to_notify = 45000`, `show_notifications = true` | zeigt die Laufzeit des letzten Befehls und löst nach 45 Sekunden eine Desktop-Benachrichtigung aus, sofern die Starship-Version dies unterstützt (TODO: prüfen, ob die per `install.sh` installierte Version Benachrichtigungen mitbringt) |
| `[time]` | `disabled = true` | keine Uhrzeit im Prompt |
| `[character]` | `❯` grün bei Erfolg, rot bei Fehler | Statusanzeige des letzten Befehls |

Der Abschnitt `[docker_context]` ist definiert, aber **nicht** in `format` aufgeführt (`$docker_context` fehlt). Das Docker-Segment erscheint daher nicht. Wer es sehen will, ergänzt `$docker_context\` in `format` vor `$conda`. Ebenso zeigt der Prompt nur die in `format` aufgeführten Sprachsegmente (`c`, `rust`, `golang`, `nodejs`, `php`, `java`, `kotlin`, `haskell`, `python`).

Die Vorlage verwendet Symbole der Nerd Font (Powerline- und Geräte-Symbole). Ohne diese Schrift erscheinen im Terminal Kästchen oder Fragezeichen, siehe [Debugging und Fehlersuche](debugging.md).
