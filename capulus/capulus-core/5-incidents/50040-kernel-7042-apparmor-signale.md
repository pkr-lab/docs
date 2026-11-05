# Incident-Report — 05.11.2026: Kernel 7.0.0-42 verweigert Signale in Containern, Zammad-Datenbank startet nicht mehr

| | |
|---|---|
| **Betroffen** | Beide k3s-Server (`homeserver` und `controlplane-1`). Sichtbar ausgefallen: Zammad (Datenbank und Init-Job). Zusätzlich verweigerte AppArmor Signale für `frr-k8s`, `gosu`, `tini`, `helm`, `coredns`, `node_exporter` und die MDM-Programme `scepserver` und `depserver` |
| **Ausfall** | Zammad seit dem Neustart von `homeserver` um 21:03, vollständig seit dem Neustart des Postgres-Pods um ca. 22:11. Zum Zeitpunkt dieses Textes ist der Server `homeserver` noch nicht zurückgestellt |
| **Datenverlust** | Keiner bekannt. Die WAL-Wiederherstellung von Postgres lief bis `redo done` durch. Ob Zammad nach der Behebung vollständig ist, wird danach geprüft |
| **Ursache** | Der Kernel `7.0.0-42` (seit dem Neustart von `homeserver` um 21:03 und von `controlplane-1` um ca. 20:46 in Betrieb, vorher `7.0.0-38`) verweigert über AppArmor das Senden von Signalen zwischen Prozessen desselben Containers |
| **Status** | `controlplane-1` läuft wieder auf `7.0.0-38` und ist verifiziert. `homeserver` läuft noch auf `7.0.0-42`, das Zurücksetzen steht aus (Abschnitt [Offene Punkte](#offene-punkte)) |

---

## Kurzfassung

Nach dem planmäßigen Neustart von `homeserver` um 21:03 hing der Zammad-Init-Job in `Init:1/2`, Abfragen auf die Tabelle `translations` standen minutenlang auf `ExecuteGather`, und der Autovacuum-Launcher von Postgres meldete jede Minute `autovacuum worker took too long to start; canceled`. Als der Postgres-Pod um ca. 22:11 neu angelegt wurde, scheiterte die Wiederherstellung am Ende: `FATAL: could not signal for checkpoint: Permission denied`. Die Liveness-Probe beendete den Container, und der Pod lief in eine Schleife.

Die Ursache fand sich im Kernel-Log: AppArmor verweigert unzählige Signale (`apparmor="DENIED" operation="signal"`, Absender `cri-containerd.apparmor.d`, Empfänger `cri-containerd.apparmor.d//&unconfined`). Postgres braucht Signale zwischen seinen Prozessen (Checkpointer, Autovacuum, parallele Worker) und läuft ohne sie nicht. Mit dem Neustart war ein neuerer Kernel gestartet, der schon Tage vorher installiert, aber nie gebootet worden war. Der Kernel `7.0.0-38` zeigt das Verhalten nicht.

---

## Systemüberblick

| Baustein | Aufgabe | Relevanz |
|---|---|---|
| **AppArmor** | Standardmäßig wendet containerd das Profil `cri-containerd.apparmor.d` auf jeden Container an | Ort der Verweigerung |
| **runc** | Startet die Container und wechselt dabei das AppArmor-Profil | Im Audit-Log steht `change_profile unprivileged unconfined converted to stacking` beim Start |
| **Kernel 7.0.0-38 / 7.0.0-42** | Beide installiert, GRUB startete mit `GRUB_DEFAULT=0` immer den neuesten | `7.0.0-42` verweigert die Signale |
| **unattended-upgrades** | Installiert Kernel-Updates, startet aber nicht neu | Der neue Kernel blieb unbemerkt, bis zum nächsten Neustart |
| **Zammad-Postgres** | Bitnami-Image, Checkpointer und Autovacuum brauchen Signale | Sichtbarer Ausfall |

---

## Zeitleiste

Alle Zeiten CET am 05.11.2026, Postgres-Logs stehen in GMT (eine Stunde früher).

| Zeit | Ereignis |
|---|---|
| ca. 20:46 | `controlplane-1` startet im Rahmen des Neustarttests neu und läuft ab jetzt mit `7.0.0-42` |
| 21:01 bis 21:03 | Planmäßiger Neustart von `homeserver`. Das Journal zeigt `poweroff.target`, der Host ist um 21:02:56 wieder oben, jetzt mit `7.0.0-42` |
| ab 21:03 | Zammad: `zammad-init-*` bleibt in `Init:1/2`, Postgres meldet jede Minute `autovacuum worker took too long to start; canceled`, Abfragen auf `translations` hängen |
| 21:15 bis 22:08 | Suche: Ressourcen (PIDs, Speicher, `/dev/shm`) sind unauffällig, `pg_stat_activity` zeigt Abfragen auf `ExecuteGather` seit über einer Stunde, keine blockierte Sperre |
| ca. 22:11 | Der Postgres-Pod wird neu angelegt. Wiederherstellung: `redo starts at 3/277BEF70`, `redo done`, dann `could not signal for checkpoint: Permission denied`. Die Liveness-Probe beendet den Container, Neustartschleife |
| 22:14 | `journalctl -k` zeigt die Verweigerungen (`operation="signal"`, `comm="postgres"`, `comm="frr-k8s"`). In einer Stunde 2155 für `postgres`, 721 für `frr-k8s`, dazu `bundle`, `helm`, `gosu`, `tini`, `coredns`, `scepserver`, `depserver`, `node_exporter` |
| 22:17 | etcd-Snapshot `vor-kernel-rollback` als Rückfallpunkt |
| ca. 22:25 | `controlplane-1` startet einmalig mit `7.0.0-38` (`grub-reboot`). Danach 4 Verweigerungen in 15 Minuten, alle vom Hostprogramm `who` (Lesezugriffe auf `/etc/passwd`), keine Container-Signale. Signaltest in einem Pod: `SIGNAL_OK` |
| ca. 22:30 | Gegenprobe auf `homeserver` (`7.0.0-42`): Signaltest in einem Pod liefert `SIGNAL_FEHLER` |

---

## Diagnose

### Was der Fehler im Container bedeutet

Ein Prozess im Container kann einem anderen Prozess im selben Container kein Signal schicken. Der einfache Test schlägt unter `7.0.0-42` fehl, unter `7.0.0-38` nicht:

```bash
kubectl run sigtest-$RANDOM --rm -i --restart=Never --image=busybox:1.36 \
  --overrides='{"spec":{"nodeName":"<node>"}}' \
  --command -- sh -c 'sleep 30 & pid=$!; kill -INT $pid && echo SIGNAL_OK || echo SIGNAL_FEHLER'
```

### Wie die Verweigerung aussieht

```
apparmor="DENIED" operation="signal" class="signal" profile="cri-containerd.apparmor.d"
comm="postgres" requested_mask="send" denied_mask="send" signal=int
peer="cri-containerd.apparmor.d//&unconfined"
```

Absender und Empfänger sind derselbe Container, der Empfänger trägt aber das gestapelte Label `…//&unconfined`. Die Signalregel des Standardprofils passt auf dieses Label nicht. Beim Start der Container schreibt das Audit-Log `change_profile unprivileged unconfined converted to stacking`. Diese Deutung der Zeilen ist eine Auslegung und nicht von AppArmor oder dem Kernel-Hersteller bestätigt. Belegt sind der Zusammenhang mit dem Kernel und das Fehlen unter `7.0.0-38`.

### Wie man es prüft

```bash
uname -r
sudo journalctl -k --since "-1h" --no-pager | grep -c 'apparmor="DENIED"'
sudo journalctl -k --since "-1h" --no-pager | grep 'apparmor="DENIED"' | grep -o 'comm="[^"]*"' | sort | uniq -c | sort -rn | head
```

Gesund sind wenige Verweigerungen von Hostprogrammen (`who`), keine von Containern. Hunderte von Container-Programmen (`postgres`, `frr-k8s`) sind das Fehlerbild.

---

## Behebung

1. etcd-Snapshot als Rückfallpunkt (`sudo k3s etcd-snapshot save --name vor-kernel-rollback`).
2. Auf dem Server `GRUB_DEFAULT=saved` setzen, `update-grub`, `grub-set-default 0`, dann einmalig `grub-reboot "Advanced options for Ubuntu>Ubuntu, with Linux 7.0.0-38-generic"` und neu starten. Der einmalige Start fällt bei einem Fehlstart von selbst auf den bisherigen Kernel zurück.
3. Nach dem Start `uname -r`, die Verweigerungen und den Signaltest prüfen.
4. Erst wenn der Server sauber läuft: `grub-set-default` auf den Eintrag mit `7.0.0-38` setzen und den Kernel mit `apt-mark manual` vor dem automatischen Aufräumen schützen.
5. `controlplane-1` zuerst, `homeserver` danach, weil nur `homeserver` die Datenbanken trägt.

---

## Lehren

- **Ein Kernel-Update wirkt erst beim nächsten Neustart, oft Tage später.** `unattended-upgrades` installiert den Kernel, startet aber nicht neu. Der Fehler zeigt sich dann beim ersten, planmäßigen Neustart, wenn man ihn nicht erwartet.
- **Nach jedem Neustart prüfen**, nicht nur ob die Nodes `Ready` sind: `uname -r`, die Zahl der AppArmor-Verweigerungen und der Signaltest. Zammad fiel erst bei der Datenbank auf, alle Pods zeigten `Running`.
- **Nie beide Server mit demselben, neuen Kernel neu starten.** `controlplane-1` lief schon eine Stunde auf `7.0.0-42` und zeigte bereits 785 Verweigerungen pro Stunde, ohne dass jemand hingesehen hat. Ein neuer Kernel wird zuerst auf `controlplane-1` gestartet und geprüft, erst danach auf `homeserver`.
- **Die hängende parallele Abfrage und die Autovacuum-Warnung waren der erste Hinweis.** Beide sprechen für Prozesse, die sich nicht gegenseitig erreichen. Das Kernel-Log war die richtige Stelle, nicht die Postgres-Konfiguration.
- **Ein Pod, den die Liveness-Probe beendet, wiederholt eine unvollständige Wiederherstellung.** Das schadet den Daten nicht, hält aber die Fehlersuche an. Die WAL-Wiederherstellung selbst war in Ordnung.
- **Der Standardkernel gehört festgelegt.** `GRUB_DEFAULT=0` bedeutet „der neueste“. Mit `GRUB_DEFAULT=saved` und einem benannten Eintrag bestimmt man selbst, wann ein neuer Kernel produktiv wird.

---

## Offene Punkte

| Punkt | Stand |
|---|---|
| `homeserver` auf `7.0.0-38` zurücksetzen | offen |
| `7.0.0-38` als Standard auf beiden Servern (`grub-set-default`, `apt-mark manual`) | offen |
| Zammad nach dem Rollback prüfen: Postgres `Ready`, Init-Job `Completed`, Oberfläche und Tickets vollständig | offen |
| Absicherung per Ansible (Standardkernel festlegen, Prüfung nach dem Neustart) | zu entscheiden |
| Prüfen, ob ein späterer Kernel das Verhalten nicht mehr zeigt, dann bewusst auf einem Server testen | offen, Beobachtung |
| Ursache beim Kernel-Hersteller melden oder nachlesen (Ubuntu-Fehlerliste, AppArmor-Stacking) | offen |
| Docker-Container auf `controlplane-1` (`tinyteller`, `day-pilot`) haben ebenfalls Signale verloren, ihr Zustand nach dem Rollback ist zu prüfen | offen |

Verwandt: [Cluster Power Manager](../../capulus-ansible/2-betrieb-hardware/20020-cluster-power-manager.md), [k3s-Referenz](../b-kubernetes-gitops/b0000-k3s.md).
