# INC-007 – TeamViewer-Dienst aktiv und aktiviert (Remote-Management-Kanal)

| Feld | Wert |
|---|---|
| ID | INC-007 |
| Datum | 2026-11-14 |
| Kategorie | botnet/apt (Remote-Zugriff) |
| Schweregrad | **Niedrig** (bei bewusster Nutzung), sonst Mittel |
| Betroffen | `teamviewerd.service`, `/etc/teamviewer/global.conf` |

## Beschreibung und Nachweis
```
$ systemctl is-active teamviewerd  -> active      $ systemctl is-enabled teamviewerd -> enabled
$ sudo ss -tulpn | grep 5939       -> 127.0.0.1:5939  users:(("teamviewerd",pid=6352))  (nur Loopback, bestätigt)
$ find /etc -mtime -7 | grep teamviewer -> /etc/teamviewer/global.conf (in den letzten 7 Tagen geändert)
$ cat /etc/teamviewer/global.conf  -> Keine Berechtigung (nur root)
```
Der Daemon baut ausgehend zu den TeamViewer-Servern auf und ermöglicht Fernsteuerung des Desktops. Ob das gewollt ist und ob unbeaufsichtigter Zugriff konfiguriert ist, ließ sich ohne root nicht klären; die Änderung von `global.conf` ist nicht per se verdächtig (TeamViewer schreibt die Datei bei Updates/Anmeldung), sollte aber bestätigt werden.

## Update 2026-11-14: Auswertung von `/etc/teamviewer/global.conf` (vom Benutzer ausgeführt)
Nur Schlüsselnamen/Struktur ausgewertet, keine Werte übernommen:
- Version **15.81.5**; `AutoServiceInstallationCoreActive=1`, `RemoteManagementServicesAutoInstallationActive=1`, `LicenseType`/`Had_a_commercial_Connection=1`.
- `DeviceManagementV2\ManagerKeys`/`ManagerMappings` mit **zwei Manager-Einträgen**: Das Gerät ist einem TeamViewer-Konto/einer Management-Konsole zugeordnet. Wer Zugriff auf dieses Konto hat, kann das Gerät verwalten.
- Das Client-Zertifikat (`X509CertificateChain`) ist vom **08.11.2026 bis 06.12.2026** gültig: Es wurde am 08.11. erneuert. Damit ist die Änderung von `global.conf` in den letzten Tagen erklärt (normale Zertifikatsrotation).
- Es gibt **keinen** Hinweis auf ein gespeichertes Gerätepasswort für unbeaufsichtigten Zugriff in den sichtbaren Schlüsseln (kein Eintrag wie `PermanentPassword`/`Always_Online` – bei verschlüsselten `[bin]`-Werten nicht sicher beurteilbar).
**Hinweis zum Umgang mit Geheimnissen:** Die Datei wurde unmaskiert in diese Sitzung eingefügt. Sie enthält privates Schlüsselmaterial (`CertificateKey`, `SK`, `X509CertificateKey` u. a.) sowie die Geräte-ID. Diese Werte stehen damit im Sitzungsverlauf (und ggf. in dessen Protokollen). Ich habe sie nirgends gespeichert. Empfehlung: Gerät in der TeamViewer-Konsole entfernen und neu anmelden bzw. `sudo teamviewer --daemon stop`, die Datei zurücksetzen und den Daemon neu starten, damit neue Schlüssel entstehen. Künftig `sed 's/=.*/=<hidden>/'` verwenden.

## Risiko
Ein kompromittiertes TeamViewer-Konto oder ein schwaches Gerätepasswort gäbe Dritten vollständige Desktop-Kontrolle ohne SSH-Spuren. Dasselbe Muster nutzen Angreifer zur Persistenz.

## Empfohlener Fix
1. Bewusst entscheiden, ob der Zugang nötig ist. Wenn nein: Skript ausführen.
2. Wenn ja: 2FA am TeamViewer-Konto, „Unbeaufsichtigter Zugriff“ nur mit starkem Passwort, Zugriff auf Verbindungspartner-Whitelist beschränken, Dienst nur bei Bedarf starten (`systemctl disable teamviewerd`, manuell `start`).
3. Optional (sudo, lesend): `sudo cat /etc/teamviewer/global.conf | sed 's/=.*/=<hidden>/'` zur Prüfung der Schlüsselnamen.

## Fix-Skript
[`scripts/incident-fixes/INC-007-teamviewer-deaktivieren.sh`](https://github.com/pkr-lab/capulus-core/blob/main/scripts/incident-scripts/incident-fixes/INC-007-teamviewer-deaktivieren.sh), sichert `/etc/teamviewer` und den Dienstzustand.

## Rollback
`sudo systemctl enable --now teamviewerd.service`; Konfiguration aus `~/.incident-backups/INC-007-*/teamviewer`.

## Quellen
- https://www.teamviewer.com/en/trust-center/security/
- https://www.cisa.gov/news-events/news/securing-remote-access-software
