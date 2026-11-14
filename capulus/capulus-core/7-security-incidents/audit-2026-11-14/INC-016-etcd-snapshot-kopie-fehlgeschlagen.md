# INC-016 – etcd-Snapshot-Kopie auf das NAS schlägt auf control-plane-1 fehl (NFS-Mount defekt)

| Feld | Wert |
|---|---|
| ID | INC-016 |
| Datum | 2026-11-14 |
| Kategorie | ransomware (Backup/Resilienz) |
| Schweregrad | **Mittel** (Homeserver kopiert weiterhin; sonst Hoch) |
| Betroffen | `control-plane-1` (192.168.178.95): `mnt-k3s\x2detcd\x2dsnapshots.mount`, `k3s-etcd-snapshot-copy.service`; NAS `192.168.178.97` |

## Beschreibung und Nachweis
```
$ systemctl --failed   (control-plane-1)
* mnt-k3s\x2detcd\x2dsnapshots.mount     loaded failed failed /mnt/k3s-etcd-snapshots
* k3s-etcd-snapshot-copy.service         loaded failed failed Copy k3s etcd snapshots to the NAS
$ journalctl -u mnt-k3s\x2detcd\x2dsnapshots.mount
2026-11-12T18:33:48 Mounting … /mnt/k3s-etcd-snapshots
2026-11-12T18:33:49 mount.nfs: Network is unreachable for 192.168.178.97:/volume1/k8s-storage/k3s-etcd-snapshots
2026-11-12T18:33:49 Failed to mount … (status=32)
$ journalctl -u k3s-etcd-snapshot-copy.service
2026-11-14T06:20:18 Failed with result 'exit-code'    2026-11-14T12:20:18 Failed (status=1/FAILURE)
$ systemctl cat mnt-… -> Type=nfs, Options=nofail,_netdev,x-systemd.mount-timeout=30
```
Beim Boot am 12.11. um 18:33 war das Netzwerk noch nicht bereit; der Mount wurde danach **nicht erneut versucht**, seit 2 Tagen schlägt jede 6-Stunden-Kopie fehl. Das Skript sendet offenbar eine Benachrichtigung (ntfy-Zeile im Journal, Inhalt nicht ausgewertet). Auf dem Homeserver ist derselbe Timer fehlerfrei (keine fehlgeschlagene Unit). Ob auf dem NAS bereits aktuelle Snapshots der Control-Planes liegen, wurde nicht geprüft. Lokale Snapshot-Verzeichnisse waren ohne root nicht lesbar.

## Risiko
Fällt ein Control-Plane aus oder verschlüsselt Ransomware den Knoten, fehlt für control-plane-1 die externe Kopie des etcd-Stands der letzten Tage. Die Sicherung ist nicht „offline/unveränderlich“, wenn das NAS die Dateien beschreibbar einbindet.

## Empfohlener Fix
1. Trockenlauf: `ssh control-plane-1 'bash -s -- --dry-run' < scripts/incident-fixes/INC-016-etcd-snapshot-nfs-mount.sh`
2. Ausführen (das Skript nutzt `sudo`, der Benutzer `ubuntu` hat dort `NOPASSWD`): `ssh control-plane-1 'bash -s' < scripts/incident-fixes/INC-016-etcd-snapshot-nfs-mount.sh`. Es legt ein Drop-in `After=/Wants=network-online.target` für die Mount-Unit an, mountet neu und startet den Kopierdienst.
3. Prüfen: NAS erreichbar (`ping 192.168.178.97`), Export `/volume1/k8s-storage/k3s-etcd-snapshots` vorhanden, neuester Snapshot auf dem NAS jünger als 6 h.
4. Empfehlung: Snapshots auf dem NAS per Synology-Snapshot-Replikation oder Immutability (WORM) schützen und den Kopierdienst per Monitoring-Alarm überwachen.

## Fix-Skript
[`scripts/incident-fixes/INC-016-etcd-snapshot-nfs-mount.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-016-etcd-snapshot-nfs-mount.sh) – idempotent, Backup von `/etc/fstab` und Drop-in unter `/var/backups/incident-INC-016-*`.

## Rollback
`sudo rm -r /etc/systemd/system/<mount-unit>.d && sudo systemctl daemon-reload` (Original aus dem Backup), `sudo umount /mnt/k3s-etcd-snapshots`.

## Quellen
- https://docs.k3s.io/datastore/backup-restore
- https://www.freedesktop.org/software/systemd/man/systemd.mount.html
