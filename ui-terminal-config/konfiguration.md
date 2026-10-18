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
