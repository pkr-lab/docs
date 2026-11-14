# INC-013 – Hardware-/Systemprüfung der Ziel-PCs/VMs: teilweise durchgeführt (Adressbereich geklärt)

| Feld | Wert |
|---|---|
| ID | INC-013 |
| Datum | 2026-11-14 |
| Kategorie | hardware |
| Schweregrad | **Info** (Prüfstatus) |
| Betroffen | angegeben: 192.157.178.94–100; laut `~/.ssh/config`: 192.168.178.94–100 |

## Beschreibung und Nachweis
```
$ ssh -p 22 -o BatchMode=yes -o ConnectTimeout=8 ubuntu@192.157.178.94 … .100
-> "Connection timed out" für alle sieben Adressen (nichts erreicht, kein Login, keine Daten)
```
192.157.178.x ist ein **öffentlicher** Bereich, der in deiner Konfiguration nicht als eigenes Netz erkennbar ist. Die Timeouts erfolgten beim reinen Login-Versuch. `~/.ssh/config` definiert dagegen genau diese Hosts im privaten Bereich: `control-plane-0 (.94)`, `control-plane-1 (.95)`, `worker-1 (.96)`, `prod-vm (.99)`, `entw-vm (.100)` jeweils mit User `ubuntu`. Für `.97` und `.98` gibt es keinen Eintrag. Ich vermute einen Tippfehler (157 statt 168). Der Versuch, die 192.168.178.x-Adressen zu prüfen, wurde vom Berechtigungssystem blockiert, weil er von deiner wörtlichen Angabe abweicht. Ich habe das nicht umgangen.

## Update 2026-11-14: Prüfung nach Rückfrage durchgeführt
Du hast auf „das scannt meinen PC und nicht den Homeserver“ hingewiesen; deine UFW-Regel nennt 192.168.178.94 „Homeserver“, damit ist der private Bereich 192.168.178.94–100 bestätigt (die öffentliche 192.157.178.x wurde nur initial per Login-Versuch berührt, Timeout, keine Daten). Ein einzelner Login-Test auf .94 wurde danach zugelassen, ebenso die Tests auf die weiteren Hosts. Es wurden ausschließlich **lesende Kommandos ohne `sudo`** ausgeführt; Rohdaten liegen unter `evidence/`.

| Adresse | Host | Status |
|---|---|---|
| 192.168.178.94 | homeserver (HP ProBook, k3s, KVM) | geprüft (ohne root) – INC-016…021 |
| 192.168.178.95 | control-plane-1 (Lenovo M720q, k3s) | geprüft (ohne root) |
| 192.168.178.96 | worker-1 | **nicht erreichbar** (`No route to host`; im Tailnet seit 24 Tagen offline) |
| 192.168.178.97 | `ugreen-nas` (UGREEN NAS laut Inventar, UGOS) | nicht geprüft (kein SSH-Zugang konfiguriert); Ziel der etcd-Snapshots, siehe INC-016 |
| 192.168.178.98 | unbekannt | nicht geprüft (kein Eintrag in `~/.ssh/config`) |
| 192.168.178.99 | prod-vm (KVM-Gast auf dem Homeserver) | geprüft (ohne root) |
| 192.168.178.100 | entw-vm | **nicht erreichbar** (`No route to host`) |

Noch offen auf den Servern: alles, was `sudo` braucht (`ufw status`, `ss -p`, `dmidecode`, `smartctl` direkt, `boltctl`, Konfigdateien in `/etc/cups`). Die Nutzer haben dort `NOPASSWD`-sudo; ich führe sie erst nach deiner ausdrücklichen Freigabe aus, je Befehl mit Erklärung.

## Risiko
Ursprünglich (vor dem Update) waren die Server-Checks nicht durchgeführt worden. Weiterhin ungeprüft: Intel AMT/ME-Status (nur über `sudo`/MEI-Tools), IPMI/BMC (Consumer-Hardware ohne BMC; auf dem Homeserver kein `ipmi`-Gerät sichtbar), NIC-Firmware-Versionen und `boltctl`.

## Empfohlener Fix
Worker-1 und entw-vm einschalten (oder bewusst als abgeschaltet dokumentieren) und dann erneut prüfen. Die NAS (.97) und .98 benennen/zuordnen. Danach führe ich ausschließlich lesende Kommandos aus. Hinweis: Bei VMs (prod-vm, entw-vm, worker-1, control-planes) sind BIOS-/SMART-/Thunderbolt-Checks teils sinnlos (virtualisierte Hardware); sinnvoll sind dort Kernel, Mitigations, Pakete, Ports, Nutzer, Cron, authorized_keys, Backups. Für `sudo` auf den Zielen nenne ich dir jeden Befehl vorher.

## Fix-Skript
Keines.

## Rollback
Entfällt.

## Quellen
- `~/.ssh/config` (Hostdefinitionen), IANA-Zuteilung 192.157.0.0/16 (öffentlich) vs. RFC 1918 (192.168.0.0/16)
