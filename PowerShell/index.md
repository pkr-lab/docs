# PowerShell – Skriptsammlung für Windows-Administration

Diese Sammlung automatisiert wiederkehrende Aufgaben auf Windows‑Systemen (z. B. Windows‑Updates).  
Ziel ist eine **klare Dokumentation**, **nachvollziehbare Systemänderungen** und **sichere** Ausführung.

> ⚠️ **Wichtige Hinweise**
> - Verwende Skripte **zuerst in einer Testumgebung**.
> - Lesen: Abschnitt **„Was wird am PC verändert?“** pro Skript.
> - Führe Skripte, die Systemkomponenten anfassen, **mit Administratorrechten** aus.

---

## Inhaltsverzeichnis

- [Voraussetzungen](#voraussetzungen)
- [Sicherheit & Verantwortungsbereich](#sicherheit--verantwortungsbereich)
- [Skriptkatalog](#skriptkatalog)
  - [`1020115_Win_UpdateScript.ps1`](#1020115_win_updatescriptps1)
- [Allgemeine Ausführung](#allgemeine-ausführung)
- [Protokollierung (Logging)](#protokollierung-logging)
- [Troubleshooting](#troubleshooting)
- [Rollback / Wiederherstellung](#rollback--wiederherstellung)
- [Vorlage für neue Skripte](#vorlage-für-neue-skripte)
- [Lizenz](#lizenz)
- [Mitwirken](#mitwirken)
- [Changelog](#changelog)

---

## Voraussetzungen

- Windows 10 oder Windows 11 (Client); lokale Administratorrechte für Systemeingriffe
- PowerShell 5.1 oder PowerShell 7.x
- Internetzugang, wenn Updates oder Module aus Onlinequellen bezogen werden
- Ggf. temporäres Umgehen der Execution Policy im aktuellen Prozess:
  ```powershell
  powershell.exe -ExecutionPolicy Bypass -File .\DeinSkript.ps1

  
Sicherheit & Verantwortungsbereich

Prüfe Quellcode und Parameter vor Produktionseinsatz.
Dokumentiere geplante Änderungen und mögliche Nebenwirkungen.
Lege Wiederherstellungsoptionen fest (Systemwiederherstellung, Backups, Treiber‑Rollbacks, etc.).
Nutze gestaffelte Rollouts (Pilot → breiter Rollout).


Skriptkatalog

Aktueller Stand: Das Repository enthält derzeit ein Skript. Weitere Skripte können jederzeit ergänzt werden (siehe Vorlage).

1020115_Win_UpdateScript.ps1
Zweck
Automatisiert das Suchen, Herunterladen und Installieren von Windows‑Updates auf dem lokalen System. Eignet sich für manuelle Wartungsfenster, On‑Demand‑Patching oder als Baustein in wiederkehrenden Update‑Jobs.
Typische Einsatzszenarien

Schnelles Schließen von Sicherheitslücken außerhalb des regulären Patchdays
Vorbereitete Update‑Läufe mit optionalem automatischem Neustart
Wartung von Geräten, die (noch) nicht zentral über WSUS/WUfB/Intune gepatcht werden

Funktionsweise (High‑Level)

Startet eine Updatesuche über Windows Update und listet verfügbare Pakete.
Lädt Updates herunter und installiert sie in einem Durchlauf.
Abhängig von der Implementierung können zwei Ansätze genutzt werden:

über das Windows Update COM‑API (Microsoft.Update.Session), oder
über das Community‑Modul PSWindowsUpdate (Cmdlets wie Get‑WindowsUpdate, Install‑WindowsUpdate).


Optional: detaillierte Laufzeit‑Ausgaben, Protokollierung in Logdateien, erzwungener Neustart (wenn Updates es verlangen oder ein Parameter dies anweist).

Voraussetzungen (skriptspezifisch)

PowerShell mit Adminrechten
Internetzugang oder Erreichbarkeit des eigenen WSUS (falls konfiguriert)
Falls das Skript PSWindowsUpdate nutzt: Modul muss installiert und importiert werden
(Beispiel – nur falls im Skript wirklich so vorgesehen):
  Install-Module PSWindowsUpdate -Scope AllUsers -Force
  Import-Module PSWindowsUpdate  
PowerShellInstall-Module PSWindowsUpdate -Scope AllUsers -ForceImport-Module PSWindowsUpdateWeitere Zeilen anzeigen


Parameter (Beispiele – abhängig von der konkreten Implementierung)

-AcceptAll – akzeptiert alle gefundenen Updates ohne Rückfrage
-AutoReboot – führt nach Bedarf automatisch einen Neustart aus
-Verbose – ausführliche Ausgabe
-WhatIf – Trockenlauf ohne tatsächliche Installation (nur wenn im Skript implementiert)


Bitte die genauen Parameter, Standardwerte und Pflichtparameter der realen Implementierung dem Skriptkopf/Kommentarblock entnehmen. Trage sie unten in der Parameter‑Tabelle nach, sobald final.

Was wird am PC verändert?

Installation von Windows‑Updates: neue Binaries, Sicherheits‑ und Qualitätsupdates; ggf. optionale Treiber/Firmware (falls ausdrücklich eingeschlossen).
Neustart: falls erforderlich (Kernel/Servicing) oder per Parameter erzwungen.
Logs: Erzeugt/aktualisiert Logdateien (siehe Abschnitt Protokollierung).
Keine weiteren dauerhaften Änderungen (z. B. Registry/Scheduled Tasks), außer diese sind ausdrücklich im Skript implementiert.

Risiken & Nebenwirkungen

Inkompatibilitäten durch einzelne Updates sind möglich (selten, aber nicht ausgeschlossen).
Ein automatischer Neustart kann laufende Sessions/Anwendungen beenden – daher zuvor Benutzer informieren oder Wartungsfenster nutzen.
Treiber-/Firmware‑Updates nur nach Geräte‑Kompatibilität testen/zulassen.

Beispiele (sofern unterstützt)
  # Interaktiver Lauf mit ausführlicher Ausgabe
  .\1020115_Win_UpdateScript.ps1 -Verbose
  
  # Vollautomatischer Lauf inkl. Reboot (falls erforderlich)
  .\1020115_Win_UpdateScript.ps1 -AcceptAll -AutoReboot
  
  # Trockenlauf (zeigt nur an, was passieren würde)
  .\1020115_Win_UpdateScript.ps1 -WhatIf

Typischerweise werden Logs in einem Ordner wie C:\Temp\Logs\ oder unter C:\Windows\Logs\WindowsUpdate\ erstellt/aktualisiert (genauen Pfad bitte aus dem Skript übernehmen).
Zusätzlich kann -Verbose genutzt werden, um Konsole/Transkript zu füllen.

Rollback / Wiederherstellung (skriptspezifisch)

Unmittelbar nach Installation: Problematische Updates über „Installierte Updates“ (Systemsteuerung/Einstellungen) oder via PowerShell/WSUS wieder deinstallieren.
Treiber/Firmware: Gerätemanager → Treiber „Vorheriger Treiber“ (falls verfügbar).
Systemzustand: Wiederherstellungspunkt/Backup (falls vorab erstellt).

Kompatibilität

Windows 10/11 (Client); Server‑Betriebssysteme nur, wenn explizit vorgesehen/getestet.
Falls WSUS oder Windows Update for Business Richtlinien aktiv sind, richtet sich die Verfügbarkeit mancher Updates danach.

Bekannte Einschränkungen

Offline‑Geräte ohne Internet/WSUS erreichen keinen Katalog.
Einige Updates erfordern mehrere Durchläufe (Suchen → Installieren → Reboot → Nachsuchen).


Allgemeine Ausführung

PowerShell als Administrator starten.
In den Repository‑Ordner wechseln.
Gewünschtes Skript mit Parametern ausführen (siehe Beispiele).
Hinweise/Prompts beachten; ggf. Neustart einplanen.


Protokollierung (Logging)

Wenn im Skript implementiert: Erstellung/Append von Logfiles (z. B. unter C:\Temp\Logs\...).
Windows‑Update‑Standardprotokolle können zusätzlich herangezogen werden (z. B. WindowsUpdate.log – je nach Windows‑Version ggf. über Get-WindowsUpdateLog generiert).


Troubleshooting

„Es wurden keine Updates gefunden“

Internet/WSUS‑Erreichbarkeit prüfen; Gruppenrichtlinien (WUfB/WSUS) verifizieren.


„PSWindowsUpdate nicht gefunden“ (falls verwendet)

Modul mit Adminrechten installieren und importieren (siehe oben).


„Execution Policy blockiert“

Signierte Skripte bevorzugen; alternativ temporär -ExecutionPolicy Bypass für diesen Start verwenden.


Fehlercodes & Logs

Konsole/Transkript prüfen, Skript‑Logs sichten, anschließend Windows‑Update‑Protokolle auswerten.




Rollback / Wiederherstellung

Einzelnes Update entfernen (Einstellungen/Systemsteuerung oder WSUS).
Treiber‑Rollback über Gerätemanager (falls unterstützt).
Systemweite Wiederherstellung nur, wenn im Vorfeld Wiederherstellungspunkt/Backup angelegt wurde.


Vorlage für neue Skripte
Nutze diese Struktur für jedes weitere Skript:
Skriptname.ps1

Zweck:
Funktionsweise (High‑Level):
Voraussetzungen:
Parameter (mit Defaults):
Was wird am PC verändert?:
Risiken & Nebenwirkungen:
Beispiele:
Protokollierung:
Rollback:
Kompatibilität & Einschränkungen:


Bitte bei jedem Skript explizit alle Systemänderungen dokumentieren (Registry, Dienste, Aufgabenplanung, Dateien/Ordner, Netzwerkziele, Firewall‑Regeln etc.), damit die Auswirkungen jederzeit nachvollziehbar sind.


Lizenz
Sofern nicht anders angegeben: MIT (oder hier deine Wunschlizenz eintragen).

Mitwirken
Pull Requests und Issues sind willkommen.
Bitte beschreibe Änderungen am Verhalten/Parameter‑Set und dokumentiere alle Systemänderungen im README‑Abschnitt des jeweiligen Skripts.
