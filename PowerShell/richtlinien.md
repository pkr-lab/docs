# Richtlinien – Windows-Policy- und GPO-Konfiguration

Skripte zur Konfiguration von Windows-Richtlinien via Registry (ADMX-konform).
Werden automatisch vom Watchdog bei jedem Login ausgeführt.

> ⚠️ Die meisten Skripte benötigen **Administratorrechte** (HKLM-Schreibzugriff).

## Dateien

| Skript | Zweck | Admin? |
|--------|-------|--------|
| `10020115_Win_R_Passwort.ps1` | PIN-Komplexität (Windows Hello) | Ja |
| `10020115_Win_R_Personalisierung.ps1` | Desktop- und Sperrbildschirm-Hintergrund sperren | Ja |
| `10020115_Win_R_SoftwareInstallUninstall.ps1` | Software-Richtlinien zurücksetzen (Rollback) | Ja |
| `10020115_Win_R_SystemHilfe.ps1` | OEM-Branding und Legal Notice | Ja |
| `10020115_Win_R_WinInstallStore.ps1` | Microsoft Store deaktivieren | Ja |

---

## `10020115_Win_R_Passwort.ps1`

**Zweck:** Setzt PIN-Komplexitätsregeln für Windows Hello via Registry.

**Was wird geändert:**
- `HKLM:\SOFTWARE\Policies\Microsoft\PassportForWork\PINComplexity`
- Mindestlänge: 8 Zeichen
- Ablauf: 180 Tage
- Groß-/Kleinbuchstaben, Ziffern, Sonderzeichen: Pflicht

```powershell
.\10020115_Win_R_Passwort.ps1
```

---

## `10020115_Win_R_Personalisierung.ps1`

**Zweck:** Setzt Organisations-Hintergründe für Desktop und Sperrbildschirm und sperrt Benutzern das Ändern.

**Was wird geändert:**
- `HKCU\...\Policies\Explorer`, `\System`, `\ActiveDesktop`
- `HKLM:\SOFTWARE\Policies\Microsoft\Windows\Personalization`
- Startet Explorer neu (damit Hintergrund sofort sichtbar)

**Bildpfade:** `C:\Users\Public\10020115_WinScripts\Win11_C\Bilder\Hintergrund\`

---

## `10020115_Win_R_SoftwareInstallUninstall.ps1`

**Zweck:** Rollback – entfernt alle Software-Beschränkungsrichtlinien und stellt Windows-Standard wieder her.

**Was wird entfernt:**
- SRP (Software Restriction Policies) aus `HKLM`
- `SettingsPageVisibility`, `BlockRemoval`, `NoAddRemovePrograms`, `DisableProgramsControlPanel` aus `HKCU`

---

## `10020115_Win_R_SystemHilfe.ps1`

**Zweck:** OEM-Branding (Systemeigenschaften → Support-Info) und Legal Notice am Anmeldebildschirm.

**Was wird geändert:**
- `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\OEMInformation` (Hersteller, Modell, Support)
- `HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon` (Legal Notice)
- `HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters` (Computerbeschreibung)

---

## `10020115_Win_R_WinInstallStore.ps1`

**Zweck:** Deaktiviert den Microsoft Store per Gruppenrichtlinie.

**Was wird geändert:**
- `HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore\RemoveWindowsStore = 1`

**Rollback:**
```powershell
Remove-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\WindowsStore" -Name "RemoveWindowsStore"
```
