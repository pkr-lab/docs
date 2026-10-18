# Installation

## Voraussetzungen

- **PowerShell 7+** oder **Windows PowerShell 5.1** (bereits auf jedem Windows vorinstalliert) – beide werden gleichwertig unterstützt, keine Installation und keine Admin-Rechte nötig
- **Unterstützte Plattformen**: Windows, macOS, Linux (PowerShell 7+); Windows PowerShell 5.1 nur unter Windows
- **Keine externen Module nötig**: Die Konfiguration liegt als PowerShell-Data-Datei (`.psd1`) vor und wird mit dem eingebauten `Import-PowerShellDataFile` gelesen (verfügbar seit PowerShell 5.0) – dadurch funktioniert alles rein mit Bordmitteln, ganz ohne `Install-Module`

### PowerShell installieren (nur für macOS/Linux nötig - optional)

Auf dem eigentlichen Zielsystem (Windows-Schulrechner) ist **nichts zu installieren**:
Windows PowerShell 5.1 ist auf jedem Windows bereits vorhanden, DocFlowEngine läuft
damit ohne jeden zusätzlichen Schritt. Die folgenden Befehle sind nur für alle, die
DocFlowEngine testweise unter macOS/Linux ausführen möchten (z. B. Entwicklung) -
PowerShell 5.1 selbst existiert dort nicht, PowerShell 7+ ist der Ersatz.

```bash
# macOS mit Homebrew (kein sudo nötig)
brew install powershell

# Linux (Ubuntu/Debian) - benötigt sudo/Admin-Rechte für die Paketinstallation;
# alternativ ganz ohne Admin-Rechte über das offizielle .tar.gz-Archiv
# (https://github.com/PowerShell/PowerShell/releases) in ein Benutzerverzeichnis entpacken
sudo apt-get install -y powershell
```

## Setup

```powershell
git clone https://github.com/pkr-lab/DocFlowEngine.git
cd DocFlowEngine
```

## Erster Lauf

1. Passe die Konfiguration in `config/docflow-config.psd1` an. Pfade werden nur ganz oben im Block `paths` geändert, alle anderen Einträge verweisen per `{name}` darauf (siehe [DocFlowEngine/konfiguration.md](konfiguration.md#paths-optional-empfohlen---pfade-nur-ganz-oben-anpassen)).
2. Führe das Skript im DryRun-Modus aus, um die geplanten Aktionen zu prüfen:
   ```powershell
   pwsh -NoProfile -ExecutionPolicy Bypass -File ./scripts/DocFlowEngine.ps1 -DryRun
   # oder mit Windows PowerShell 5.1:
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\DocFlowEngine.ps1 -DryRun
   ```
3. Wenn alles passt, führe es ohne `-DryRun` aus:
   ```powershell
   pwsh -NoProfile -ExecutionPolicy Bypass -File ./scripts/DocFlowEngine.ps1
   # oder mit Windows PowerShell 5.1:
   powershell.exe -NoProfile -ExecutionPolicy Bypass -File .\scripts\DocFlowEngine.ps1
   ```
