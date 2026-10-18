# Nutzung

## Typische Abläufe

| Aufgabe | Befehl |
|---|---|
| Watchdog einrichten | `.\Ausführung\WatchDogInstall.ps1` (als normaler Benutzer) |
| Einzelnes Richtlinien-Skript testen | `powershell.exe -ExecutionPolicy Bypass -File .\Richtlinien\10020115_Win_R_Firewall.ps1` (als Administrator) |
| Updates im Hintergrund starten | `.\Update\10020115_Win_UpdateScript.ps1 -RUN_IN_BACKGROUND` |
| Skripte in eine Test-VM kopieren und GPO exportieren | `.\Ausführung\DateienSendenVM.ps1` |
| SSH zu einem Mac per MAC-Adresse | `.\SSH_toMac.ps1 -TargetMac <MAC>` |

Jedes Skript und jeder Ordner ist in einer eigenen Seite beschrieben, siehe [Überblick](index.md#überblick).

## Troubleshooting

| Problem | Lösung |
|---------|--------|
| Skript wird blockiert | `-ExecutionPolicy Bypass` verwenden |
| PSWindowsUpdate nicht gefunden | `Install-Module PSWindowsUpdate -Force` als Admin |
| Watchdog startet nicht | `WatchDogInstall.ps1` als normaler Benutzer (nicht als Admin) ausführen |
| Logs leer / nicht vorhanden | Pfad `C:\Users\Public\10020115_WinScripts\Logs\` manuell anlegen |
| OEM-Branding nicht sichtbar | Windows neu starten |
| WLAN-Meldebox erscheint nicht | Zähldatei prüfen: `C:\Users\Public\10020115_WinScripts\10020115_Win_R_WLANCounter.txt` |

## Rollback

| Aktion | Befehl |
|--------|--------|
| Watchdog-Task entfernen | `Unregister-ScheduledTask -TaskName "GPO-Watchdog" -Confirm:$false` |
| Run-Key entfernen | `Remove-ItemProperty "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name GPOWatchdogStarter` |
| Store-Richtlinie entfernen | `Remove-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" -Name RemoveWindowsStore` |
| PIN-Richtlinie entfernen | `Remove-Item "HKLM:\SOFTWARE\Policies\Microsoft\PassportForWork\PINComplexity" -Recurse -Force` |
| Software-Richtlinien | `.\Richtlinien\10020115_Win_R_SoftwareInstallUninstall.ps1` |
| Legal Notice entfernen | Registry: `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` → `LegalNoticeCaption` + `LegalNoticeText` löschen |
| OEM-Branding entfernen | `Remove-Item "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\OEMInformation" -Recurse` |
