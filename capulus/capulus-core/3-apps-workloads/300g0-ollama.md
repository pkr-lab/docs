# Ollama — LLM auf dem Desktop-PC, per Wake-on-LAN geweckt

> **Ort:** Desktop-PC `ollama-pc` (RTX 3080 Ti, 12 GB VRAM, 64 GB RAM, Ubuntu 26.04 Desktop) · Aufrufer: n8n im TECH-Cluster · Code: `argocd/apps/tech/n8n/ollama-pc/` in capulus-core und die Rolle [`ollama_pc`](../../capulus-ansible/rollen/ollama_pc.md) in capulus-ansible. `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

Ollama liefert die KI-Antwortentwürfe für E-Mails und Tickets. Es läuft nicht mehr im Cluster, sondern auf dem Desktop-PC. Der PC schläft, bis ein Request kommt: Der Homeserver weckt ihn per Wake-on-LAN, wartet auf Ollama und ruft dann direkt dessen API auf. Qualität geht vor Geschwindigkeit, eine Antwort darf bis zu 15 Minuten dauern. Genutzt wird Ollama vom n8n-Workflow „Zammad Tickets verarbeiten“ ([30070](30070-n8n.md#beispiel-workflow-externer-zammad--ki-antwortentwurf-ollama)).

Begründungen und Fallstricke zum Code: [60040](../6-hintergruende/60040-helm-charts-tech.md#ollama).

---

## Architektur

```
n8n (Zammad Tickets verarbeiten, je Ticket)
  1. LLM-Eingabe bauen            Verlauf, Mail, Zusatzkontext als JSON (Base64)
  2. cli.js probe                 GET <PC>:11434/api/tags, Timeout 3 s
  3. PC schläft?  ja ->  POST carplay-api/api/power/wake {"target":"ollama-pc"}
                           carplay-api -> power-agent (Host homeserver) -> wakeonlan <MAC>
  4. cli.js chat                  wartet mit Backoff bis Ollama antwortet (max. 120 s),
                                  lädt das Modell vor (leerer /api/chat, keep_alive),
                                  baut den Prompt, kürzt den Verlauf, ruft /api/chat auf
  5. Antwortentwurf extrahieren   -> interne Notiz in Zammad, ntfy-Push
```

| Baustein | Ort | Aufgabe |
|---|---|---|
| `cli.js` und `lib/*.js` | `argocd/apps/tech/n8n/ollama-pc/`, als ConfigMap `n8n-ollama-pc` nach `/opt/ollama-pc` im n8n-Pod | Wake-and-Call-Logik: `probe`, `wake`, `chat`, `ps` |
| `prompts/email_system.md`, `prompts/email_examples.md` | ebenda, ConfigMap, `/opt/ollama-pc/prompts` | Systemprompt und Beispiel-Antworten |
| `knowledge/*.md` | ebenda, ConfigMap, `/opt/ollama-pc/knowledge` | Firmenwissen, bei jeder Anfrage neu gelesen |
| `benchmark.js`, `test-wake-and-chat.sh`, `testdata/emails/` | ebenda, nicht in der ConfigMap | Modellvergleich und End-to-End-Test von Hand |
| power-agent auf dem Homeserver | Rolle [`power_agent`](../../capulus-ansible/rollen/power_agent.md) | sendet das Magic Packet (`wakeonlan`), Ziel `ollama-pc` |
| Rolle `ollama_pc` | capulus-ansible, Playbook `ollama-pc.yml` | Ollama, systemd-Override, WoL, Resume-Hook, GPU, UFW |

**Warum geht das Magic Packet nicht direkt aus dem n8n-Pod?** Ein Pod hängt im Overlay-Netz, ein Broadcast ins Heimnetz verlässt es nicht zuverlässig. Der power-agent läuft auf dem Host des Homeservers und sendet ihn im LAN. Der Token für carplay-api liegt als n8n-Credential „carplay-api API Token“, deshalb ist der Wake ein eigener HTTP-Node im Workflow und nicht Teil des Skripts (Wake-Modus `none` im Pod).

**Wake-Modi des Skripts** (`OLLAMA_WAKE_MODE`): `udp` (Magic Packet direkt, für Aufrufe von einem Host im LAN wie Benchmark und Test), `http` (POST an einen Relay wie carplay-api), `command` (beliebiges Kommando, z. B. `wakeonlan <MAC>`), `none` (nur pollen, der Aufrufer hat bereits geweckt).

**Keine Egress-NetworkPolicy nötig.** Der Namespace `n8n` hat keine Egress-Regeln, nur die generische `tier-default-ingress`. Der Aufruf `n8n → PC:11434` geht deshalb ohne Änderung. Eine zusätzliche Egress-Policy würde den gesamten übrigen Egress von n8n sperren (Zammad, ntfy, DNS) und ist hier der falsche Weg. Die Absicherung liegt auf dem PC: UFW lässt Port 11434 nur vom Homeserver zu (die Pods erscheinen mit der Node-IP des Homeservers).

---

## Modellwahl und Begründung

| | |
|---|---|
| Standard | `gemma3:27b-it-q8_0` |
| Vergleich | `llama3.3:70b` |
| Umschalten | `OLLAMA_MODEL` (n8n-Chart: `ollamaPc.env`), ohne Codeänderung |

- **Hardware:** 12 GB VRAM (rund 11,5 GB nutzbar, der Desktop belegt ca. 750 MB) und 64 GB RAM (rund 52 GB frei). Beide Modelle passen nicht in den VRAM. Ollama lagert Schichten in den RAM aus, das ist ausdrücklich gewollt.
- **`gemma3:27b-it-q8_0`:** 27B-Modell in 8 Bit, rund 29 GB. Es ist der beste Kompromiss aus Antwortqualität (Q8 verliert kaum gegenüber 16 Bit) und Speicherbedarf. Es passt in den RAM mit viel Reserve, ein Teil der Schichten liegt auf der GPU.
- **`llama3.3:70b`:** Standard-Tag in 4 Bit, rund 43 GB. Größeres Modell, deutlich mehr Schichten im RAM, entsprechend langsamer. Es dient dem Qualitätsvergleich mit dem Benchmark ([unten](#benchmark)).
- **Der tatsächliche GPU-Anteil und die Token/s stehen nicht fest, bevor man sie misst.** Der Benchmark gibt sie je Modell aus (`ollama ps`, `eval_count`, Token/s).
- **Kontext 16384 Token** (`OLLAMA_CONTEXT_LENGTH` auf dem PC, `num_ctx` im Request): lange E-Mail-Verläufe passen, der KV-Cache bleibt klein genug. `OLLAMA_KV_CACHE_TYPE=q8_0` und `OLLAMA_FLASH_ATTENTION=1` halbieren den KV-Cache-Bedarf gegenüber 16 Bit.
- **`OLLAMA_KEEP_ALIVE=30m`:** Das Modell bleibt nach einer Antwort 30 Minuten geladen, damit mehrere Tickets hintereinander nicht jedes Mal neu laden.
- **`temperature` 0,6:** genug Variation für natürliche Formulierungen, wenig Halluzination.

Die frühere Cluster-Variante (`llama3.2:3b` auf worker-0, Pentium G5400T ohne AVX2) schaffte rund 1 Token/s und scheiterte an größeren Modellen am Speicher. Das war die harte Grenze, nicht die Konfiguration.

---

## Setup Desktop-PC

Voraussetzungen: Ubuntu 26.04 mit installiertem NVIDIA-Treiber (`nvidia-smi` funktioniert), LAN-Kabel, feste DHCP-Reservierung. Die BIOS-Einstellungen stehen im [nächsten Abschnitt](#bios-einstellungen), sie sind Voraussetzung für Wake-on-LAN und gehören nicht in die Rolle.

### Mit Ansible (empfohlen)

Die Rolle `ollama_pc` richtet alles ein und ist wiederholbar. Der PC steht im Inventar als `ollama-pc` (`192.168.178.66`, Benutzer `pke`), Schnittstelle `enp7s0`, NetworkManager-Verbindung `netplan-enp7s0`, MAC `34:5a:60:50:f5:ec` (`host_vars/ollama-pc/vars.yml`).

Direkt auf dem PC, ohne SSH (fragt Vault- und sudo-Passwort ab):

```bash
make ollama-pc-local-check   # Dry-run direkt auf dem PC
make ollama-pc-local         # direkt auf dem PC
```

Per SSH vom Verwaltungsrechner: `make ollama-pc-check` (Dry-run) und `make ollama-pc`. Dafür muss auf dem PC ein SSH-Server laufen (`sudo apt install openssh-server`) und der Schlüssel des Verwaltungsrechners in `~/.ssh/authorized_keys` stehen.

Was die Rolle tut ([Details](../../capulus-ansible/rollen/ollama_pc.md)):

1. Prüft `nvidia-smi`, installiert `curl`, `zstd`, `ethtool`, `network-manager`, `ufw`.
2. Installiert Ollama mit dem offiziellen Installationsskript, falls `ollama` fehlt (`ollama_pc_version` pinnt die Version).
3. Schreibt den systemd-Override `/etc/systemd/system/ollama.service.d/override.conf` mit `OLLAMA_HOST`, `OLLAMA_KEEP_ALIVE`, `OLLAMA_FLASH_ATTENTION`, `OLLAMA_KV_CACHE_TYPE`, `OLLAMA_CONTEXT_LENGTH`.
4. Hängt `nvidia-persistenced` beim Boot ein (die Unit ist `static`, die Rolle legt den Symlink in `multi-user.target.wants` selbst an) und deployt `nvidia-power-limit.service` (`nvidia-smi -pm 1` und `-pl <Watt>`, Standard 280). Eine vorhandene gleichnamige Unit wird ersetzt.
5. Setzt `802-3-ethernet.wake-on-lan magic` per `nmcli`, aktiviert Wake-on-LAN sofort mit `ethtool -s <if> wol g` und bricht ab, wenn `ethtool` danach nicht `Wake-on: g` zeigt.
6. Legt den Resume-Hook `/usr/lib/systemd/system-sleep/ollama-resume` an: nach jedem Aufwachen werden `nvidia-power-limit.service` und `ollama.service` neu gestartet und der Leerlauf-Zähler des Auto-Suspends zurückgesetzt.
7. Richtet den [Auto-Suspend](#auto-suspend) ein (Timer `ollama-idle-suspend.timer`).
8. UFW: SSH aus dem LAN, Port 11434 nur vom Homeserver, Standardrichtlinie `deny incoming`, UFW aktivieren.
9. Startet Ollama, wartet auf `/api/tags` und lädt fehlende Modelle aus `ollama_pc_models` (Standard `gemma3:27b-it-q8_0`; für den Vergleich `llama3.3:70b` ergänzen).

**Nach dem ersten Lauf einmal neu starten** und prüfen, dass die Einstellungen den Reboot überstehen:

```bash
sudo ethtool enp7s0 | grep Wake-on      # Wake-on: g
nmcli -g 802-3-ethernet.wake-on-lan connection show netplan-enp7s0
systemctl is-active ollama nvidia-persistenced
nvidia-smi --query-gpu=persistence_mode,power.limit --format=csv
curl -s http://127.0.0.1:11434/api/tags
```

Die NetworkManager-Verbindung stammt aus Netplan (Datei unter `/run/NetworkManager/system-connections`). Ob `nmcli connection modify` die Einstellung dauerhaft über den Reboot rettet, ist nicht vorab getestet. Zeigt `ethtool` nach dem Reboot `Wake-on: d`, in der Netplan-Datei der Schnittstelle `wakeonlan: true` setzen und `sudo netplan apply` ausführen.

### Per Hand (Äquivalent, jeder Befehl braucht sudo)

Für den Fall ohne Ansible. Die Befehle ändern das System, deshalb vor dem Ausführen lesen:

```bash
curl -fsSL https://ollama.com/install.sh | sh
sudo systemctl edit ollama      # [Service]-Block mit den Environment-Zeilen aus dem Override
sudo nmcli connection modify netplan-enp7s0 802-3-ethernet.wake-on-lan magic
sudo ethtool -s enp7s0 wol g
sudo ufw allow from 192.168.178.94 to any port 11434 proto tcp
sudo ufw allow from 192.168.178.0/24 to any port 22 proto tcp
sudo ufw default deny incoming
sudo ufw enable
```

Override-Inhalt:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_KEEP_ALIVE=30m"
Environment="OLLAMA_FLASH_ATTENTION=1"
Environment="OLLAMA_KV_CACHE_TYPE=q8_0"
Environment="OLLAMA_CONTEXT_LENGTH=16384"
```

---

## BIOS-Einstellungen

Wake-on-LAN funktioniert nur, wenn das Mainboard die Netzwerkkarte im Schlafzustand mit Strom versorgt. Das ist BIOS-Sache. Der PC ist ein MSI-Board (`MS-7E59`), die Bezeichnungen unterscheiden sich je nach BIOS-Version.

| Einstellung (typischer Name bei MSI) | Wert |
|---|---|
| Wake Up Event Setup → Resume By PCI-E Device / „Power On By PCI-E“ | Enabled |
| Wake Up Event Setup → Resume By LAN / „Wake on LAN“ (falls vorhanden) | Enabled |
| ErP Ready / EuP (Advanced → Power Management Setup) | Disabled |
| Deep Sleep (falls vorhanden) | Disabled |

- **ErP muss aus sein.** Mit ErP Ready schaltet das Board die Standby-Versorgung der Netzwerkkarte ab, ein Magic Packet kommt dann nie an.
- Der Zielzustand ist **Suspend (S3)**. Aufwachen aus dem Ausschalten (S5) braucht zusätzlich „Resume By PCI-E“ und funktioniert nur mit ErP aus.
- Das Magic Packet erreicht den PC nur im selben Subnetz, über LAN-Kabel (nicht WLAN).

---

## Setup Homeserver

1. **Wake-Ziel im power-agent.** Die Rolle `power_agent` kennt `ollama-pc` als Ziel „nur wecken“ (`power_agent_wake_only_targets`). Auf dem Homeserver ausrollen:
   ```bash
   make power-agent
   ```
   Danach steht `ollama-pc` in `TARGETS` in `/etc/power-agent/config.env`. carplay-api reicht das Ziel unverändert durch (`POST /api/power/wake {"target":"ollama-pc"}`), es braucht keine Codeänderung.
2. **PC-Adresse im n8n-Chart.** `argocd/apps/tech/n8n/values.yaml`, Abschnitt `ollamaPc.env`, ist auf `OLLAMA_URL: "http://192.168.178.66:11434"` gesetzt. Ändert sich die IP des PCs, hier und in `host_vars/ollama-pc` und `inventory/hosts.yml` anpassen. Andere Werte siehe [Env-Variablen](#env-variablen).
3. **Prompts und Firmenwissen füllen.** `prompts/email_system.md` (Name, Signatur), `prompts/email_examples.md` (2–3 Beispiele) und `knowledge/firma.md` ([Firmenwissen](#firmenwissen)). Solange `[PLATZHALTER…]` im Text steht, warnt das Skript, und die Beispiele werden ignoriert. Aufbau: [Systemprompt](#aufbau-des-systemprompts).
4. **Workflows aktualisieren.** In n8n die Workflows `zammad-tickets-verarbeiten.json` und `zammad-externer-ki-lauf-taeglich.json` neu importieren (Stand 2026-10-31 per API in die bestehenden Workflows eingespielt, IDs und Aktivierung blieben) bzw. nach dem Muster in [30070](30070-n8n.md#beispiel-workflow-externer-zammad--ki-antwortentwurf-ollama) anpassen. Der Node „Ollama-PC wecken“ nutzt das bestehende Credential „carplay-api API Token“.
5. **Cluster-Ollama entfällt.** Mit dem Merge entfernt ArgoCD die Application `ollama` samt PVC (`prune`). Das Modell auf der PVC (`llama3.2:3b`) geht damit verloren, es wird nicht mehr gebraucht. Der leere Namespace `ollama` kann danach von Hand gelöscht werden: `kubectl delete namespace ollama`.
6. **Test:** [End-to-End-Test](#end-to-end-test).

---

## Zusatzkontext und Verlauf aus Zammad

Der Node „LLM-Eingabe bauen“ macht aus einem Ticket die Eingabe für `cli.js chat`:

| Feld | Quelle |
|---|---|
| `mail` | die letzte Kunden-Nachricht (`sender: Customer`), sonst die letzte öffentliche Nachricht |
| `history` | alle übrigen öffentlichen Nachrichten, chronologisch |
| `context` | interne Notizen, die mit `Kontext:` beginnen (Text nach dem Präfix) |

Interne Notizen ohne `Kontext:` und System-Nachrichten fließen **nicht** ein, damit frühere KI-Entwürfe nicht als Kontext zurückkehren. Der Payload ist auf 90 000 Byte begrenzt (Shell-Argument-Limit), älteste Mails fallen vorher weg (Log im n8n-Pod). Die genaue Kürzung auf das Token-Budget macht `cli.js`.

---

## Request an Ollama

| Schritt | Detail |
|---|---|
| Health-Check | `GET /api/tags`, Timeout `OLLAMA_HEALTH_TIMEOUT_SECONDS` (3 s), getrennt vom Request-Timeout |
| Wake | je nach Modus, danach Polling mit Backoff (2 s, ×1,5, höchstens 15 s) bis `OLLAMA_WAKE_TIMEOUT_SECONDS` (120 s) |
| Vorladen | `POST /api/chat` mit leerer `messages`-Liste, `keep_alive` und denselben `options.num_ctx`, Timeout `OLLAMA_PRELOAD_TIMEOUT_SECONDS` (900 s). Ändert sich `num_ctx` zwischen Vorladen und Anfrage, lädt Ollama das Modell neu |
| Anfrage | `POST /api/chat`, `stream: false`, `options: {num_ctx, temperature}`, `keep_alive`, Timeout `OLLAMA_REQUEST_TIMEOUT_SECONDS` (1200 s) |
| Kürzen | Der Verlauf wird von den ältesten Mails her gekürzt, bis Systemprompt, Kontext, Verlauf und Mail ins Budget passen (`num_ctx` minus `OLLAMA_RESPONSE_RESERVE_TOKENS`). Das Ergebnis steht als Warnung in der Antwort (`warnings`) und im Log. Der Prompt sagt dem Modell, dass ältere Mails fehlen |
| Nachprüfung | Meldet Ollama mehr `prompt_eval_count`, als das Budget zulässt (Schätzung zu optimistisch), wird der Verlauf einmal mit dem gemessenen Zeichen/Token-Verhältnis neu gekürzt |
| Zu große Pflichtteile | Passen Systemprompt, Kontext und Mail allein nicht ins Budget, bricht der Aufruf mit `PROMPT_TOO_LARGE` ab, statt still abzuschneiden |
| Ausgabe | nur der Antworttext. Eine führende `Betreff:`-Zeile wird entfernt |

Fehlercodes in der Ausgabe von `cli.js` (`{"ok":false,"error":{"code":…,"message":…}}`, Exit-Code 1):

| Code | Bedeutung |
|---|---|
| `PC_NOT_WAKING` | Ollama antwortet nach dem Wake nicht innerhalb des Zeitlimits |
| `WAKE_FAILED` | das Wake-Signal ließ sich nicht senden (Relay nicht erreichbar, MAC ungültig) |
| `MODEL_MISSING` | Modell nicht auf dem PC (`ollama pull <Modell>`) |
| `MODEL_LOAD_TIMEOUT`, `MODEL_LOAD_FAILED` | Vorladen schlug fehl oder dauerte zu lang |
| `REQUEST_TIMEOUT`, `CHAT_FAILED` | Anfrage zu langsam oder von Ollama abgelehnt |
| `PROMPT_TOO_LARGE` | Pflichtteile überschreiten `num_ctx` |
| `EMPTY_REPLY` | Modell lieferte keinen Text |
| `INVALID_INPUT`, `SYSTEM_PROMPT_MISSING` | Eingabe oder Prompt-Datei fehlerhaft |

---

## Aufbau des Systemprompts

Die Nachricht an das Modell besteht aus zwei Teilen.

**System-Nachricht** (aus Dateien, nicht im Code):

1. `prompts/email_system.md`: Rolle und Schreibstil, Anrede-Regel (Du oder Sie aus der eingehenden Mail übernehmen, im Zweifel Sie), Signatur, Faktenregel (Preise, Links und Fakten nur aus `<firmenwissen>`, sonst `[OFFEN: …]`), Ausgabe-Regeln (nur Antworttext, ohne Betreffzeile und Erklärungen)
2. `prompts/email_examples.md` als Abschnitt „Beispiel-Antworten“, mit 2–3 Beispielen (`**Eingehende E-Mail:**` / `**Antwort:**`)
3. das [Firmenwissen](#firmenwissen) als Abschnitt `<firmenwissen>…</firmenwissen>`

**User-Nachricht** (vom Skript gebaut), Abschnitte durch `---` getrennt:

1. `## Zusatzkontext`, nur wenn vorhanden
2. `## E-Mail-Verlauf (chronologisch, älteste zuerst)`, nur wenn vorhanden, mit Hinweis auf weggelassene Mails
3. `## Zu beantwortende E-Mail`
4. Schlusssatz: nur den Antworttext ausgeben

Das Firmenwissen steht am Ende der System-Nachricht und damit vor dem Mail-Verlauf, der in der User-Nachricht kommt. Eigene Dateien über `OLLAMA_SYSTEM_PROMPT_FILE` und `OLLAMA_EXAMPLES_FILE`. Änderungen an den Prompts gehören ins Repo (ConfigMap), das Aktualisieren im Pod dauert nach dem ArgoCD-Sync rund eine Minute, ein Neustart ist nicht nötig.

---

## Auto-Suspend

Der PC schläft nach Leerlauf wieder ein, damit er nicht dauerhaft läuft. Der Timer `ollama-idle-suspend.timer` startet jede Minute `/usr/local/sbin/ollama-idle-suspend`. Das Skript hält einen Zeitstempel „zuletzt beschäftigt“ in `/run/ollama-idle-suspend/last_busy`. Der PC gilt als **beschäftigt**, solange eines davon zutrifft:

| Prüfung | Zutreffend, wenn |
|---|---|
| Ollama | `GET /api/ps` meldet ein geladenes Modell (läuft eine Anfrage, oder die `keep_alive`-Frist von 30 Minuten ist noch nicht abgelaufen) |
| Sitzung | eine logind-Sitzung hat `IdleHint=no` (jemand arbeitet am Desktop) |
| Sperre | ein Programm hält eine blockierende Sperre gegen `sleep` oder `idle` (`systemd-inhibit --list`, z. B. Download, Video) |
| Last | der 5-Minuten-Lastwert liegt bei `ollama_pc_idle_suspend_max_load` oder darüber |

Trifft nichts zu und liegt der letzte Zeitstempel mindestens `ollama_pc_idle_suspend_minutes` zurück (Standard 15), führt das Skript `systemctl suspend` aus. Zwischen der letzten Anfrage und dem Suspend vergehen damit rund 30 Minuten (`keep_alive`) plus 15 Minuten Leerlauf. Nach jedem Aufwachen setzt der Resume-Hook den Zeitstempel auf „jetzt“, sonst würde der PC zwischen Magic Packet und Modell-Laden sofort wieder einschlafen.

Ausschalten: `ollama_pc_idle_suspend_enabled: false` und die Rolle neu laufen lassen. Vorübergehend: `sudo systemctl stop ollama-idle-suspend.timer`. Protokoll: `journalctl -t ollama-idle-suspend` und `journalctl -u ollama-idle-suspend`.

Der Auto-Suspend gilt für den PC als Ganzes. Wer am Desktop arbeitet, wird nicht unterbrochen, ein langer Job ohne Eingaben, ohne Sperre und mit Last unter dem Schwellwert aber schon. Für solche Jobs `systemd-inhibit` verwenden oder `ollama_pc_idle_suspend_max_load` senken.

---

## Firmenwissen

Alles, was das Modell über die Firma wissen soll (Kontakt, Links, Preise, Regeln, häufige Fragen), steht als Markdown-Dateien in `argocd/apps/tech/n8n/ollama-pc/knowledge/`. Der Inhalt aller `*.md`-Dateien wird bei **jeder Anfrage** in die System-Nachricht eingefügt, zwischen Systemprompt (samt Beispielen) und Mail-Verlauf:

```
<firmenwissen>
### Datei: firma.md

…Inhalt…

### Datei: preise.md

…Inhalt…
</firmenwissen>
```

Das Modell darf Preise, Links, Kontaktdaten, Zeiten und Regeln nur von dort übernehmen. Fehlt eine Angabe, schreibt es `[OFFEN: <was fehlt>]` an die Stelle, damit der Mensch, der den Entwurf prüft, sie ergänzt. Diese Regel steht in `prompts/email_system.md`.

### Aufbau und Pflege

- **Dateien:** beliebig viele `*.md` in `knowledge/`, alphabetisch sortiert, ohne Unterordner. Andere Endungen werden ignoriert. Beispiel: `knowledge/firma.md` mit den Abschnitten Kontakt und Links, Preise (Tabelle), Regeln, Häufige Fragen. Eine Datei je Thema (`preise.md`, `faq.md`, `regeln.md`) hält die Pflege übersichtlich.
- **Schreibstil:** kurze, eindeutige Sätze und Tabellen. Jeder Preis mit Einheit und „netto“ oder „brutto“. Links vollständig. Keine Dopplungen, sonst widersprechen sich die Dateien irgendwann.
- **Platzhalter:** Zeilen mit `[PLATZHALTER…]` werden **nicht** an das Modell gegeben (und mit einer Warnung gemeldet). So stehen unausgefüllte Beispielzeilen nie als „Fakten“ im Prompt. Eine Zeile ist erst gültig, wenn der Platzhalter ersetzt ist. Das gilt je Zeile, eine teilweise ausgefüllte Datei wirkt also sofort mit den fertigen Zeilen.
- **Wirkung ohne Neustart:** Das Skript liest die Dateien bei jedem Aufruf neu. Im Cluster liegen sie als ConfigMap im n8n-Pod, nach einer Änderung im Repo und dem ArgoCD-Sync dauert es rund eine Minute, ein Neustart ist nicht nötig. Auf dem Rechner (Benchmark, Test) wirkt eine Änderung sofort. Anderer Ordner: `OLLAMA_KNOWLEDGE_DIR`.
- **Prüfen:** `cli.js chat` und der Benchmark schreiben bei jedem Aufruf `Firmenwissen: <n> Datei(en), ca. <x> Token (<y>% von num_ctx)` ins Log und die Warnungen ins Feld `warnings`.

### Größenlimit

Das Firmenwissen belegt Platz im Kontext (`num_ctx`, Standard 16384 Token) und fehlt dann dem Mail-Verlauf. Das Skript schätzt die Größe (3 Zeichen je Token) und **warnt, wenn es mehr als 40 % von `num_ctx` belegt** (`OLLAMA_KNOWLEDGE_WARN_RATIO`, Standard `0.4`). Bei 16384 Token sind das rund 6500 Token, etwa 20 000 Zeichen oder 8–10 Seiten Text. Die Warnung ist nur ein Hinweis, der Aufruf läuft weiter. Passen Systemprompt, Firmenwissen, Kontext und Mail zusammen nicht mehr ins Budget, bricht er mit `PROMPT_TOO_LARGE` ab. Der Verlauf wird zuerst gekürzt.

Weitere Grenzen:

- **ConfigMap:** Alle Dateien (Skripte, Prompts, Wissen) zusammen dürfen höchstens 1 MiB haben (Kubernetes-Limit). Das reicht für das Firmenwissen weit über das Token-Limit hinaus.
- **Geschwindigkeit:** Jedes Token Firmenwissen muss bei jeder Anfrage ausgewertet werden (`prompt_eval`). Bei einem Modell, das zum Teil im RAM liegt, kostet das spürbar Zeit. Die Prompt-Auswertung steht im Benchmark (`prompt_eval (s)`, Prompt-Token).
- **Qualität:** Je mehr Text im Prompt steht, desto eher überliest das Modell Details. Wenige, klare Fakten wirken besser als ein ganzes Handbuch.

### Ab wann RAG sinnvoll ist

RAG (Retrieval-Augmented Generation) legt das Wissen in einen Index und gibt dem Modell je Mail nur die passenden Abschnitte. Es lohnt sich, wenn eines davon zutrifft:

| Signal | Bedeutung |
|---|---|
| Die Warnung „Firmenwissen belegt … % von num_ctx“ erscheint dauerhaft | Das Wissen passt nicht mehr bequem in den Prompt. `num_ctx` zu erhöhen kostet VRAM (KV-Cache) und Zeit und löst es nur kurz |
| Das Wissen umfasst deutlich mehr als rund 10 Seiten (Produkthandbücher, viele Preislisten, Vertragsbedingungen) | Der Großteil davon ist für eine einzelne Mail ohnehin irrelevant |
| Die Texte ändern sich oft oder kommen aus anderen Systemen (Wiki, Shop, Dokumentenablage) | Ein Index wird automatisch aktualisiert, Dateien in Git nicht |
| Die Antworten brauchen kundenspezifische Daten (Verträge, Bestellungen) | Das gehört per Abfrage in den Kontext, nicht in eine statische Datei |

Bis dahin ist die Datei-Lösung einfacher, nachvollziehbar (alles steht im Git) und braucht keine weitere Komponente. Vor einem RAG-Umbau zuerst prüfen, ob sich das Wissen straffen lässt. Ein späterer Umbau setzt an derselben Stelle an: `lib/knowledge.js` liefert den Text für `<firmenwissen>`, dort käme stattdessen eine Suche nach den passenden Abschnitten hin (Embedding-Modell auf dem PC, z. B. `nomic-embed-text`, Texte in Abschnitte teilen, die besten drei bis fünf Treffer je Mail einfügen).

---

## Env-Variablen

### Aufrufer (`cli.js`, `benchmark.js`; im n8n-Chart unter `ollamaPc.env`)

| Variable | Default | Bedeutung |
|---|---|---|
| `OLLAMA_URL` | – (Pflicht) | Basis-URL des PCs, z. B. `http://192.168.178.66:11434` |
| `OLLAMA_MODEL` | `gemma3:27b-it-q8_0` | Modell für `chat` |
| `OLLAMA_NUM_CTX` | `16384` | Kontextlänge, in Vorladen und Anfrage identisch |
| `OLLAMA_TEMPERATURE` | `0.6` | Temperatur |
| `OLLAMA_KEEP_ALIVE` | `30m` | `keep_alive` für Vorladen und Anfrage |
| `OLLAMA_RESPONSE_RESERVE_TOKENS` | `2048` | Platz für die Antwort im Kontext |
| `OLLAMA_CHARS_PER_TOKEN` | `3` | Schätzung für das Kürzen (konservativ für Deutsch) |
| `OLLAMA_HEALTH_TIMEOUT_SECONDS` | `3` | Timeout des Health-Checks |
| `OLLAMA_WAKE_TIMEOUT_SECONDS` | `120` | maximale Wartezeit nach dem Wake |
| `OLLAMA_WAKE_POLL_INITIAL_SECONDS` | `2` | erster Polling-Abstand (wächst um Faktor 1,5) |
| `OLLAMA_WAKE_POLL_MAX_SECONDS` | `15` | größter Polling-Abstand |
| `OLLAMA_PRELOAD_TIMEOUT_SECONDS` | `900` | Timeout für das Vorladen |
| `OLLAMA_REQUEST_TIMEOUT_SECONDS` | `1200` | Timeout der Anfrage (20 Minuten) |
| `OLLAMA_WAKE_MODE` | `udp` (im n8n-Chart `none`) | `none`, `udp`, `http`, `command` |
| `OLLAMA_WAKE_MAC` | – | MAC für `udp` |
| `OLLAMA_WAKE_BROADCAST` | `255.255.255.255` | Broadcast-Adresse für `udp` |
| `OLLAMA_WAKE_PORT` | `9` | UDP-Port für `udp` |
| `OLLAMA_WAKE_URL` | – | Relay-URL für `http` (z. B. `https://carplay-api.prod.homeserver/api/power/wake`) |
| `OLLAMA_WAKE_TARGET` | `ollama-pc` | Ziel im Body für `http` |
| `OLLAMA_WAKE_TOKEN` | – | Bearer-Token für `http` |
| `OLLAMA_WAKE_COMMAND` | – | Kommando für `command` |
| `OLLAMA_WAKE_REQUEST_TIMEOUT_SECONDS` | `10` | Timeout des Wake-Aufrufs |
| `OLLAMA_PROMPTS_DIR` | `<Skriptordner>/prompts` (im Pod `/opt/ollama-pc/prompts`) | Ordner der Prompt-Dateien |
| `OLLAMA_SYSTEM_PROMPT_FILE` | `<OLLAMA_PROMPTS_DIR>/email_system.md` | Systemprompt |
| `OLLAMA_EXAMPLES_FILE` | `<OLLAMA_PROMPTS_DIR>/email_examples.md` | Beispiel-Antworten |
| `OLLAMA_KNOWLEDGE_DIR` | `<Skriptordner>/knowledge` (im Pod `/opt/ollama-pc/knowledge`) | Ordner der Wissensdateien |
| `OLLAMA_KNOWLEDGE_WARN_RATIO` | `0.4` | Anteil von `num_ctx`, ab dem die Warnung kommt |
| `OLLAMA_BENCH_MODELS` | `gemma3:27b-it-q8_0,llama3.3:70b` | Modelle des Benchmarks |

### Test-Skript

| Variable | Default | Bedeutung |
|---|---|---|
| `OLLAMA_PC_SSH` | – (Pflicht) | `benutzer@host` für den Suspend per SSH |
| `SUSPEND_COMMAND` | `sudo -n systemctl suspend` | Kommando auf dem PC |
| `SUSPEND_WAIT_SECONDS` | `180` | wie lange auf das Einschlafen gewartet wird |
| `TEST_MAIL` | `testdata/emails/02-rueckfrage-du.json` | Eingabe für den Test |

### Ollama auf dem PC (Ansible-Variablen der Rolle `ollama_pc`)

| Variable | Default | Wird zu |
|---|---|---|
| `ollama_pc_listen` | `0.0.0.0:11434` | `OLLAMA_HOST` |
| `ollama_pc_keep_alive` | `30m` | `OLLAMA_KEEP_ALIVE` |
| `ollama_pc_flash_attention` | `1` | `OLLAMA_FLASH_ATTENTION` |
| `ollama_pc_kv_cache_type` | `q8_0` | `OLLAMA_KV_CACHE_TYPE` |
| `ollama_pc_context_length` | `16384` | `OLLAMA_CONTEXT_LENGTH` |
| `ollama_pc_gpu_power_limit_watts` | `280` | `nvidia-smi -pl` |
| `ollama_pc_idle_suspend_enabled` | `true` | Auto-Suspend einrichten |
| `ollama_pc_idle_suspend_minutes` | `15` | Leerlauf in Minuten bis zum Suspend |
| `ollama_pc_idle_suspend_max_load` | `1.0` | Lastwert (5 min), ab dem der PC als beschäftigt gilt |
| `ollama_pc_models` | `[gemma3:27b-it-q8_0]` | Modelle, die die Rolle lädt |
| `ollama_pc_ufw_allowed_sources` | Homeserver-IP | erlaubte Quellen für Port 11434 |

---

## Benchmark

`benchmark.js` schickt dieselben Test-Mails an beide Modelle und vergleicht sie. Es braucht Node.js ab Version 22 auf einem Rechner im selben LAN wie der PC (kein weiterer Install, keine Abhängigkeiten). Dasselbe Skript weckt den PC bei Bedarf selbst (`OLLAMA_WAKE_MODE=udp`).

```bash
cd argocd/apps/tech/n8n/ollama-pc
export OLLAMA_URL=http://192.168.178.66:11434
export OLLAMA_WAKE_MAC=34:5a:60:50:f5:ec
node benchmark.js                                  # beide Standardmodelle
node benchmark.js --models gemma3:27b-it-q8_0      # nur eines
node benchmark.js --emails ./meine-mails --out ./ergebnis
```

Vorher muss `llama3.3:70b` auf dem PC liegen (`ollama_pc_models` ergänzen oder `ollama pull llama3.3:70b`).

Ablauf je Modell: andere geladene Modelle entladen, Modell vorladen, `ollama ps` auslesen (GPU-Anteil, Ausgabe auf der Konsole und im Bericht), dann jede Mail aus `testdata/emails/*.json` senden. Die Mails haben das Format der Eingabe von `cli.js chat` (`context`, `history`, `mail`), eigene Mails einfach als weitere `.json`-Dateien ablegen.

Ergebnis in `benchmark-results/<Zeitstempel>/` (oder `--out`):

| Datei | Inhalt |
|---|---|
| `results.md` | `ollama ps`-Tabelle, Ergebnistabelle je Modell und Mail: GPU-Anteil, Gesamtdauer, `prompt_eval_duration`, Prompt-Token, `eval_count`, Token/s, Warnungen |
| `comparison.md` | je Mail eine Tabelle mit den Antworten der Modelle nebeneinander |
| `results.json` | alle Rohwerte |

Dauer und Token/s kommen aus der Ollama-Antwort (`eval_count`, `eval_duration`), die Gesamtdauer ist die gemessene Zeit des Aufrufs. Die Qualität der Antworten beurteilt man in `comparison.md` von Hand.

---

## End-to-End-Test

`test-wake-and-chat.sh` prüft die ganze Kette: PC in Suspend versetzen, Request vom Homeserver, aufgewacht, Modell auf der GPU, Antwort da.

```bash
export OLLAMA_URL=http://192.168.178.66:11434
export OLLAMA_PC_SSH=pke@192.168.178.66
export OLLAMA_WAKE_MODE=udp OLLAMA_WAKE_MAC=34:5a:60:50:f5:ec
./test-wake-and-chat.sh
```

Schritte: (1) PC wecken, falls er schläft, (2) per SSH Suspend auslösen, (3) warten bis Ollama nicht mehr antwortet, (4) `cli.js chat` senden und prüfen, dass der PC geweckt wurde (`woke: true`) und eine Antwort kommt, (5) `ollama ps` auswerten, das Modell muss einen GPU-Anteil über 0 haben. Das Skript endet mit `TEST BESTANDEN` oder `FEHLER: …`.

Voraussetzungen auf dem PC, jeweils mit sudo und von Hand einzurichten:

- SSH-Server und der Schlüssel des Aufrufers: `sudo apt install openssh-server`.
- Suspend ohne Passwort. Beispiel `/etc/sudoers.d/ollama-suspend` (mit `sudo visudo -f` anlegen): `pke ALL=(root) NOPASSWD: /usr/bin/systemctl suspend`. Alternativ `SUSPEND_COMMAND` auf ein Kommando setzen, das ohne Passwort geht.

Unit-Tests der Skripte (Fake-Ollama, ohne PC und GPU): `cd argocd/apps/tech/n8n/ollama-pc && npm test`.

---

## Troubleshooting

| Symptom | Ursache und Prüfung |
|---|---|
| `PC_NOT_WAKING` | Der PC wacht nicht auf oder Ollama startet nicht. Zuerst `ethtool enp7s0 \| grep Wake-on` am PC (`g`), dann BIOS ([oben](#bios-einstellungen)). Den Wake einzeln testen: `wakeonlan 34:5a:60:50:f5:ec` auf dem Homeserver. Kommt das Paket an, aber Ollama nicht: `systemctl status ollama` am PC, `journalctl -u ollama -b` |
| `WAKE_FAILED` oder Wake-Node meldet 401/502 | Credential „carplay-api API Token“ ungültig oder power-agent nicht erreichbar. `systemctl status power-agent` am Homeserver, `ollama-pc` in `/etc/power-agent/config.env` unter `TARGETS`? |
| `OLLAMA_URL ist nicht gesetzt` | `ollamaPc.env.OLLAMA_URL` im n8n-Chart leer |
| `REQUEST_TIMEOUT` | Antwort länger als `OLLAMA_REQUEST_TIMEOUT_SECONDS` (1200 s). Bei `llama3.3:70b` und langen Mails möglich: Timeout erhöhen, `num_ctx` oder Verlauf verkleinern, Token/s im Benchmark prüfen |
| `MODEL_LOAD_TIMEOUT` oder Modell lädt nicht | Beim ersten Laden liest Ollama bis zu 43 GB von der Platte. `OLLAMA_PRELOAD_TIMEOUT_SECONDS` erhöhen. `journalctl -u ollama -f` am PC zeigt Fehler wie „unable to allocate“ (zu wenig freier RAM: andere Programme schließen, `free -h`) |
| `MODEL_MISSING` | `ollama pull <Modell>` am PC, oder `ollama_pc_models` ergänzen und die Rolle laufen lassen |
| Läuft nur auf der CPU (`gpu_percent` 0, `ollama ps` zeigt `100% CPU`) | `nvidia-smi` am PC. Nach einem Suspend geht die GPU für Ollama oft verloren, der Resume-Hook startet `ollama.service` neu. Hilft das nicht: `sudo systemctl stop ollama && sudo rmmod nvidia_uvm && sudo modprobe nvidia_uvm && sudo systemctl start ollama`. `journalctl -u ollama -b \| grep -i -E "cuda\|gpu"` zeigt, ob eine GPU erkannt wurde |
| Nur ein Teil des Modells auf der GPU | Erwartet bei 12 GB VRAM, siehe [Modellwahl](#modellwahl-und-begründung). Der Anteil steht im Benchmark. Bei fast vollem VRAM durch den Desktop lädt Ollama weniger Schichten, dann Browser und Co. schließen |
| Antwort bricht ab („Antwort wurde durch das Kontextlimit abgeschnitten“) | `OLLAMA_RESPONSE_RESERVE_TOKENS` erhöhen oder `OLLAMA_NUM_CTX` erhöhen (der VRAM-Bedarf des KV-Caches steigt) |
| Warnung „ältere Mails im Verlauf gekürzt“ | Kein Fehler, der Verlauf war länger als das Token-Budget. Wer mehr Verlauf will: `OLLAMA_NUM_CTX` erhöhen |
| `PROMPT_TOO_LARGE` | Systemprompt, Zusatzkontext oder die Mail selbst sind zu lang für `num_ctx`. Kontext kürzen oder `OLLAMA_NUM_CTX` erhöhen |
| Warnung „Firmenwissen belegt … % von num_ctx“ | Wissen kürzen, `OLLAMA_NUM_CTX` erhöhen oder [RAG](#ab-wann-rag-sinnvoll-ist) einsetzen |
| Warnung „Firmenwissen … Zeile(n) mit [PLATZHALTER…] ignoriert“ | Platzhalter in `knowledge/*.md` durch echte Angaben ersetzen, solange fehlt die Angabe im Prompt |
| Entwürfe enthalten `[OFFEN: …]` | Die Angabe fehlt in `<firmenwissen>`: in `knowledge/` ergänzen. Die Stelle im Entwurf ergänzt der Prüfer von Hand |
| Warnung „Systemprompt enthält noch [PLATZHALTER…]“ | Signatur und Name in `prompts/email_system.md` ausfüllen, Beispiele in `prompts/email_examples.md` ersetzen |
| Leistung oder Takt nach dem Aufwachen anders | `nvidia-power-limit.service` neu starten (der Resume-Hook tut das), `nvidia-smi -q -d POWER` prüfen |
| `make ollama-pc` bricht beim Wake-on-LAN-Check ab | NIC oder Treiber unterstützt Magic Packet nicht, oder `ollama_pc_interface` zeigt auf die falsche Schnittstelle (nicht die Bridge) |
| Nach dem Reboot `Wake-on: d` | Netplan überschreibt die NetworkManager-Einstellung: in der Netplan-Datei `wakeonlan: true` setzen |
| Der PC schläft ein, obwohl jemand arbeitet, oder nie ein | `journalctl -t ollama-idle-suspend`, `loginctl show-session <id> -p IdleHint` (Sitzung), `systemd-inhibit --list`, `curl -s localhost:11434/api/ps` (Modell geladen?) und `cat /proc/loadavg` zeigen, welche Prüfung greift |
| Der PC ist per SSH nicht erreichbar | Auf dem PC läuft kein SSH-Server (Standard bei Ubuntu Desktop), `make ollama-pc-local` nutzen |

> **Stand der Verifikation:** Die Skripte sind mit Unit-Tests und gegen einen simulierten, schlafenden PC getestet. Noch nicht gelaufen sind die Rolle `ollama_pc` auf dem echten PC, der Wake-Weg über carplay-api und power-agent sowie die geänderten n8n-Workflows im Cluster. Vor dem produktiven Einsatz einmal [den Test](#end-to-end-test) und einen manuellen Workflow-Lauf mit einem Test-Ticket ausführen.
