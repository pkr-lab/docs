# Debugging und Fehlersuche

Code: [install.sh](https://github.com/pkr-lab/ui-terminal-config/blob/main/install.sh), [starship-toml-template](https://github.com/pkr-lab/ui-terminal-config/blob/main/starship-toml-template)

Das Projekt hat kein Log und keinen Dienst. Alle Ausgaben stehen im Terminal, `install.sh` kennzeichnet sie mit `[OK]`, `[SKIP]`, `[INFO]` und `[ERROR]`. Zum genauen Nachverfolgen hilft `bash -x ./install.sh` (zeigt jeden Befehl). Das projektübergreifende Kapitel ist die [zentrale Fehlersuche](../capulus/capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

---

## Meldungen von `install.sh`

| Meldung | Ursache | Lösung |
|---|---|---|
| `[SKIP] starship (already installed)` | Starship ist im `PATH` | kein Fehler. Die Version aktualisiert man mit `curl -sS https://starship.rs/install.sh \| sh -s -- --yes --bin-dir ~/.local/bin` |
| `[SKIP] starship.toml (already linked)` | Symlink zeigt schon auf die Vorlage | kein Fehler |
| `[INFO] Backing up existing starship.toml to …` | eine eigene Konfiguration lag vor | Sicherung liegt neben der Datei, bei Bedarf zurückkopieren |
| `[INFO] npm global prefix (…) isn't writable; switching to /home/…/.local` | globales npm-Verzeichnis nicht beschreibbar | Hinweis: Das Skript setzt `npm config set prefix ~/.local`. Rückgängig mit `npm config delete prefix` |
| `[ERROR] <befehl> -- npm not found, install Node.js first` | `npm` fehlt, aber eine KI-CLI wurde angefordert | `sudo apt install nodejs npm`, Skript erneut starten. Das Skript läuft nach dieser Meldung weiter, die CLI fehlt aber |
| `[SKIP] claude (set INSTALL_CLAUDE=true to install)` (analog `codex`, `gemini`, `agy`) | Variable steht auf `false` | gewollt, oder Variable auf `true` setzen |
| `[SKIP] <befehl> (already installed)` | Programm ist bereits im `PATH` | kein Fehler |
| `[INFO] Installing unzip (needed to unpack the font) ...` | `unzip` fehlt | Skript nutzt `sudo apt-get install -y unzip`, fragt ggf. nach dem sudo-Passwort |
| `[SKIP] JetBrainsMono Nerd Font already installed` | `fc-list` findet die Schrift | kein Fehler |
| `[OK] Done.` | alles durchgelaufen | – |

Abbruch ohne eigene `[ERROR]`-Meldung (das Skript läuft mit `set -euo pipefail`):

| Symptom | Ursache | Lösung |
|---|---|---|
| `curl: (6) Could not resolve host` oder `curl: (22)` | kein Netz, Proxy oder URL nicht erreichbar | Verbindung prüfen. Die Installer kommen von `starship.rs`, `github.com/ryanoasis/nerd-fonts` und `antigravity.google` |
| `sudo: a password is required` | `unzip` soll installiert werden, aber sudo ohne Terminal | in einem Terminal starten oder `unzip` vorher installieren |
| `unzip: cannot find or open …` oder Abbruch beim Download der Schrift | Download unvollständig | Skript erneut ausführen, ein temporärer Fehler wird dadurch behoben |
| `npm ERR! code EACCES` | Berechtigungen im npm-Verzeichnis | Skript neu starten, es stellt den Präfix selbst auf `~/.local`, falls das Verzeichnis nicht beschreibbar ist |
| Abbruch bei `npm install -g …` | Paket nicht gefunden oder npm-Fehler | `npm install -g <paket>` direkt ausführen und die Fehlermeldung lesen |

---

## Fehlerbilder im Terminal

| Symptom | Ursache | Lösung |
|---|---|---|
| Prompt unverändert | Starship nicht in der Shell aktiviert | `eval "$(starship init bash)"` in `~/.bashrc`, neue Shell öffnen |
| `starship: command not found` | `~/.local/bin` nicht im `PATH` | `export PATH="$HOME/.local/bin:$PATH"` in `~/.bashrc` |
| Kästchen oder `?` statt Symbolen im Prompt | Terminal nutzt nicht die Nerd Font | Schrift `JetBrainsMono Nerd Font Mono` im Terminal einstellen, `fc-list \| grep -i "JetBrainsMono.*Nerd"` muss einen Treffer liefern |
| Symbole sehen versetzt aus | Schrift ohne „Mono“-Variante | `JetBrainsMono Nerd Font Mono` wählen |
| Schrift in `fc-list` vorhanden, Terminal zeigt sie nicht | Schrift-Cache veraltet | `fc-cache -f ~/.local/share/fonts`, Terminal neu starten |
| Segment fehlt in großen Git-Repos (mit `STARSHIP_LOG=warn` steht `Executing command … timed out`) | Git-Abfrage braucht länger als `command_timeout` (5000 ms) | `command_timeout` in der Vorlage erhöhen |
| Verzeichnis zeigt fehlende Sprach-Segmente | Datei, die das Segment erkennt, liegt tiefer als `scan_timeout` (100 ms) erlaubt | `scan_timeout` erhöhen |
| Kein Docker-Segment | `$docker_context` fehlt in `format` | siehe [Konfiguration](konfiguration.md#weitere-einstellungen-der-vorlage) |
| `starship config` meldet unbekannte Schlüssel | neuere Starship-Version ändert Schlüssel | `starship config` bzw. `starship print-config` prüfen, [Starship-Dokumentation](https://starship.rs/config/) |
| Konfiguration wirkt nicht | Symlink zeigt ins Leere (Checkout verschoben) | `ls -l ~/.config/starship.toml`, `./install.sh` im neuen Checkout erneut ausführen |
| `claude: command not found` nach der Installation | `~/.local/bin` nicht im `PATH` (npm-Präfix wurde auf `~/.local` gesetzt) | `PATH` ergänzen |
| Alte Konfiguration wiederherstellen | – | `mv ~/.config/starship.toml.bak.<Zeitstempel> ~/.config/starship.toml` (den Symlink vorher entfernen) |

---

## Checkliste „Prompt oder Schrift funktioniert nicht“

1. `command -v starship` und `starship --version`: gefunden?
2. `ls -l ~/.config/starship.toml`: Symlink auf die Vorlage im Checkout, Ziel existiert?
3. `tail ~/.bashrc`: steht `eval "$(starship init bash)"` darin?
4. `fc-list | grep -i "JetBrainsMono.*Nerd"`: Schrift installiert?
5. Terminal-Einstellungen: ist `JetBrainsMono Nerd Font Mono` gewählt?
6. `STARSHIP_LOG=trace starship prompt 2>&1 | head` zeigt, welches Segment wie lange braucht und welche Fehler Starship sieht.
7. `bash -x ./install.sh`: welcher Schritt bricht ab?

Allgemeines zur Installation: [Installation](installation.md).
