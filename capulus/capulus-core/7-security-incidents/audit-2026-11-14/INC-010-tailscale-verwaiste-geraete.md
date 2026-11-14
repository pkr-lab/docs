# INC-010 – Lange offline befindliche Geräte im Tailnet

| Feld | Wert |
|---|---|
| ID | INC-010 |
| Datum | 2026-11-14 |
| Kategorie | apt (laterale Bewegung) |
| Schweregrad | **Niedrig** |
| Betroffen | Tailscale-Netz `it.dlrg.jugend.andernach@` |

## Beschreibung und Nachweis
```
$ tailscale status
100.110.81.85  github-runnervmlun5p  offline, last seen 55d ago
100.124.213.94 worker-1              tagged-devices  offline, last seen 24d ago
100.105.105.92 infotafel             offline, last seen 1d ago
Health check: Some peers are advertising routes but --accept-routes is false
$ tailscale status   (vom Homeserver aus, zusätzlich)
100.68.141.122 github-runnervmlun5p-1  tagged-devices  offline, last seen 55d ago
```
Damit existieren **zwei** GitHub-Runner-Knoten (`…lun5p` am Arbeitsplatz-Tailnet-Blick, `…lun5p-1` mit Tag `tagged-devices` aus Sicht des Homeservers), beide seit 55 Tagen offline.
Ein GitHub-Runner-Knoten (`github-runnervm…`) ist seit 55 Tagen offline, aber weiterhin Mitglied des Netzes (vermutlich ein temporärer CI-Knoten, der nicht abgemeldet wurde). Ob er noch gültige Schlüssel besitzt und welche ACLs gelten, ist ohne Zugriff auf die Tailscale-Admin-Konsole nicht prüfbar.

## Abgleich mit dem Incident-Register (2026-11-14)
Das ist im Register der offene Incident **036** (Tailscale-ACL: Kiosk-Geräte und ein alter Runner haben Vollzugriff; Status *Offen*, Empfehlung: Runner entfernen, ephemere getaggte Knoten, Key Expiry). Neu gegenüber dem Stand vom 18.10.: `github-runnervmlun5p` ist jetzt 55 Tage offline (damals 26); der getaggte Knoten `github-runnervmlun5p-1` (in Regel 3 von 036 genannt) ist ebenfalls 55 Tage offline. Die ACL liegt in der Tailscale-Konsole, nicht im Repo.

## Risiko
Verwaiste Knoten mit gültigem Node-Key sind ein Einstiegspunkt: Wer den Runner-Schlüssel (z. B. aus einem Image/Log) besitzt, hat Netzzugriff gemäß ACL.

## Empfohlener Fix (manuell)
1. https://login.tailscale.com/admin/machines öffnen, `github-runnervmlun5p` löschen, Key-Expiry für Ephemeral-Nodes aktivieren (Auth-Keys mit „Ephemeral“ erzeugen).
2. ACLs prüfen: Tags (`tagged-devices`) nur auf notwendige Ziele (SSH 22, 6443) beschränken.
3. `worker-1` (24 Tage offline) klären: abgeschaltet oder ausgefallen?

## Fix-Skript
Keines (Löschen von Geräten und ACLs erfolgt in der Tailscale-Admin-Konsole; die CLI kann fremde Geräte nicht entfernen).

## Rollback
Gerät neu mit `tailscale up` anmelden.

## Quellen
- https://tailscale.com/kb/1111/ephemeral-nodes
- https://tailscale.com/kb/1018/acls
