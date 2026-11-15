# Rolle `cups_print_server`

Druckserver, QPDL-Treiber-Build, zweite Warteschlange.

| | |
|---|---|
| Läuft gegen | controlplane-0 |
| Tags | `cups-print-server`, `printer` |
| Quellcode | [roles/cups_print_server](https://github.com/pkr-lab/capulus-ansible/tree/main/roles/cups_print_server) |

## Variablen

Defaults aus `defaults/main.yml`. Überschreiben in `group_vars/` oder `host_vars/`.

| Variable | Default | Bedeutung |
|---|---|---|
| `cups_print_server_enabled` | `true` | – |
| `cups_print_server_printer_name` | `Samsung_M2026` | – |
| `cups_print_server_printer_description` | `Samsung Xpress M2026 (USB, homeserver)` | – |
| `cups_print_server_printer_location` | `2.OG Homeserver` | – |
| `cups_print_server_device_uri` | `` | – |
| `cups_print_server_ppd` | `` | – |
| `cups_print_server_media_size` | `A4` | – |
| `cups_print_server_set_default` | `true` | – |
| `cups_print_server_remote_admin` | `true` | – |
| `cups_print_server_remote_any` | `true` | – |
| `cups_print_server_share_printers` | `true` | – |
| `cups_print_server_packages` | `[cups, cups-client, cups-ipp-utils, printer-driver-splix, hplip, avahi-daemon, usbutils]` | – |
| `cups_print_server_disabled_services` | `[cups-browsed.service]` | Dienste, die gestoppt und deaktiviert werden (CUPS, SANE und Avahi bleiben bewusst, siehe [038](../../capulus-core/7-security-incidents/038-unnoetige-netzwerkdienste-auf-den-hosts.md)) |
| `cups_print_server_admin_users` | `[ubuntu]` | – |
| `cups_print_server_build_qpdl_driver` | `false` | – |
| `cups_print_server_qpdl_repo` | `https://gitlab.com/ScumCoder/splix.git` | – |
| `cups_print_server_qpdl_repo_version` | `f97086c367d926dc4b6f84facabc9c3029729cba` | – |
| `cups_print_server_qpdl_build_dir` | `/usr/local/src/splix-qpdl` | – |
| `cups_print_server_extra_queues` | `[]` | – |
| `cups_print_server_tls_names` | `[homeserver, homeserver.local, homeserver.fritz.box, logs-write.tech.homeserver, ubuntu…` | – |
| `cups_print_server_tls_ip` | `{{ network_static_ip \| default('192.168.178.94') }}` | – |

## Ablauf

1. CUPS-/Drucker-Pakete installieren
2. Admin-Nutzer zur lpadmin-Gruppe hinzufügen
3. cups + avahi-daemon aktivieren und starten
4. nicht benötigte cups-dienste stoppen und deaktivieren
5. Warten, bis cupsd auf Port 631 antwortet
6. Aktuelle cupsd-Einstellungen auslesen
7. CUPS-Sharing/Remote-Zugriff setzen
8. Aktuelle Subject-Alternative-Names des cupsd-TLS-Zertifikats auslesen
9. TLS-Zertifikat fuer cupsd mit allen bekannten Selbst-Identifikationsnamen erzeugen (siehe docs/2-betrieb-hardware/20040-printer.md)
10. temporaeres verzeichnis fuer das schluesselpaar anlegen (`tempfile`)
11. Temporaeres Schluessel-/Zertifikatspaar generieren
12. Zertifikat unter allen Selbst-Identifikationsnamen ablegen (`0644`)
13. Key unter allen Selbst-Identifikationsnamen ablegen (`0600`)
14. Temporaere Zertifikatsdateien entfernen (`always`, auch nach Fehlern)
15. bestehende tls-schluessel auf modus 0600 setzen
16. Build-Abhängigkeiten für den QPDL-Treiber installieren
17. QPDL-Treiber-Quellcode klonen (gepinnter Commit)
18. Patch: PPDFile::getDefaultChoice() deklarieren (ppdfile.h)
19. Patch: PPDFile::getDefaultChoice() implementieren (ppdfile.cpp)
20. Patch: Printer::loadInformation() erzwingt PPD-PageSize (printer.cpp)
21. QPDL-Filter kompilieren
22. QPDL-Filter + PPDs installieren
23. Install-Ausgabe des QPDL-Treibers anzeigen
24. Handler sofort ausführen, damit lpinfo -m die neue PPD sieht
25. Verfügbare Drucker-Device-URIs ermitteln
26. Erkannte Device-URIs anzeigen
27. Verfügbare PPDs ermitteln
28. Samsung/SPL-passende PPDs anzeigen
29. Warnen, wenn Device-URI/PPD noch nicht gesetzt sind
30. Druckerwarteschlange anlegen/aktualisieren
31. Warteschlange aktivieren
32. Job-Annahme für Warteschlange aktivieren
33. Warteschlange als System-Default setzen
34. Extra-Queues: Splix-Katalog nach passenden PPDs durchsuchen (Debug)
35. Extra-Queues: warnen, wenn ppd noch nicht gesetzt ist
36. Extra-Queues: Warteschlange anlegen/aktualisieren
37. Extra-Queues: Warteschlange aktivieren
38. Extra-Queues: Job-Annahme aktivieren
39. Extra-Queues: als System-Default setzen (falls gewünscht)
40. Status aller Warteschlangen anzeigen
41. Warteschlangen-Status ausgeben

## Handler

- Restart cups

---

## Fehlersuche

Allgemeines Vorgehen bei Rollen: `ansible-playbook <playbook>.yml --limit <host> --tags <tag> --check --diff -vvv` zeigt Änderungen ohne sie anzuwenden (`make <ziel>-check` bei den Playbooks mit Dry-Run). Die Rollenübersicht mit Make-Zielen: [Rollen](index.md#fehlersuche-je-rolle).

```bash
sudo systemctl status cups avahi-daemon
lpstat -t
sudo journalctl -u cups -n 100 --no-pager
sudo tail -50 /var/log/cups/error_log
lpinfo -v; lpinfo -m | grep -i M2026
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Drucker erscheint nicht | USB-Gerät nicht erkannt oder `cups_print_server_device_uri` leer und nicht automatisch ermittelbar | `lpinfo -v`, `lsusb`, URI in `host_vars` setzen |
| Druckauftrag bleibt hängen | Filter (QPDL) fehlt oder PPD stimmt nicht | `error_log`, `lpinfo -m`, [Drucker](../2-betrieb-hardware/20040-printer.md#fehlerbehebung) |
| QPDL-Build scheitert | Build-Abhängigkeiten oder Quellstand (die Rolle patcht `ppdfile.h`, `ppdfile.cpp`, `printer.cpp`) | Task-Ausgabe der Patch- und Compile-Schritte lesen |
| Zertifikatsfehler beim Zugriff über AirPrint/IPP | Selbstsigniertes cupsd-Zertifikat enthält den genutzten Namen nicht (die Rolle erzeugt alle bekannten SAN) | Namen prüfen, Rolle erneut ausrollen |
| Drucker nicht aus dem Heimnetz oder Tailnet erreichbar | Sharing aus (`cups_print_server_remote_*`), UFW | `cupsctl`, [Firewall](common.md) |

Allgemeine Fehlersuche: [Host-Troubleshooting](../a-betriebssystem/a0040-troubleshooting.md), [zentrale Fehlersuche](../../capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

[Alle Rollen](index.md)
