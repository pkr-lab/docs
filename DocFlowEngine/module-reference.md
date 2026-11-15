# Modul- und Funktionsreferenz

`scripts/DocFlowEngine.psm1` ist das Root-Modul: Es lädt per Dot-Sourcing alle Teilmodule aus
`scripts/DocFlowEngine/` (in dieser Reihenfolge: `Common.ps1`, `Config.ps1`, `State.ps1`,
`Lock.ps1`, `Naming.ps1`, `CopyForward.ps1`, `CopyBack.ps1`, `Rename.ps1`) und stellt `Invoke-DocFlowEngine` als
öffentlichen Einstiegspunkt bereit. `scripts/DocFlowEngine.ps1` ist der dünne CLI-Wrapper
(`-ConfigPath`, `-DryRun`), den `README.md` in den Ausführungsbeispielen verwendet.

Alle unten aufgeführten Funktionen sind über `Export-ModuleMember` öffentlich, sofern nicht
„Nicht exportiert“ dabeisteht.

### Umbenannte Funktionen (Aliase als Übergang)

Drei Funktionen trugen nicht zugelassene PowerShell-Verben und lösten beim Import eine Warnung aus.
Sie heißen jetzt anders. Die alten Namen bleiben als exportierte Aliase erhalten, auch beim Import
über `DocFlowEngine.psd1` (`AliasesToExport`). Neue Aufrufe sollten die neuen Namen verwenden.

| Neuer Name | Alter Name (Alias) |
|---|---|
| `Import-DocFlowConfig` | `Load-Config` |
| `Import-DocFlowState` | `Load-State` |
| `Initialize-DocFlowTargetDirectories` | `Ensure-TargetDirectories` |

### Entfernte Funktionen

`Get-FileProject`, `Get-ProjectRoutes` und `Resolve-ProjectTarget` (Legacy-Routing über eine
`project`-Regexgruppe) und der Parameter `Copy-NewFiles -ProjectRoutes` sind entfernt.
`Invoke-DocFlowEngine` hat dieses Routing nie angeschlossen. Das Kategorie-Routing
(`Get-FileCategory` / `Resolve-CategoryTarget`) bleibt bestehen.

## `DocFlowEngine.psm1` (Root)

| Funktion | Zweck |
|---|---|
| `Invoke-DocFlowEngine -ConfigPath <string> [-DryRun]` | Hauptablauf: Config laden, Lock erwerben, Laufkontext bauen (`New-DocFlowRunContext`), `Invoke-DocFlowCopyForward` + `Copy-ReviewedFiles` ausführen, State speichern, Lock freigeben. Einziger Ort, der die Teilmodule orchestriert. |

## `Common.ps1` - Logging, Pfade, generische Hilfsfunktionen

| Funktion | Zweck |
|---|---|
| `Write-Log -Level <Trace\|Debug\|Info\|Warning\|Error> -Message <string>` | Schreibt nach `Write-Host` und (falls konfiguriert) in die Log-Datei, gefiltert nach `log.level`. Siehe [Logging.md](logging.md). |
| `Expand-Template -Template <string> -Context <hashtable>` | Ersetzt `{key}`-Platzhalter im Template durch Werte aus `Context`. |
| `Resolve-PathOrAbsolute -PathValue <string>` | Löst einen (ggf. wildcardhaltigen) Pfad zu einem absoluten Pfad auf, auch wenn er noch nicht existiert (schneidet Segmente vom Ende ab, bis ein existierender - ggf. ebenfalls wildcardhaltiger - Elternordner gefunden wird). |
| `Resolve-SourcePaths -PathValue <string>` | Löst einen Quellpfad zu allen tatsächlich existierenden, passenden Ordnern auf (`Resolve-Path`, kann bei Wildcards mehrere Treffer liefern). |
| `Test-PathExcluded -FullName <string> -ExcludePaths <array>` | Prüft, ob ein Pfad unterhalb eines der übergebenen Ausschlussordner liegt (Präfixvergleich). |
| `ConvertTo-DocFlowHashtable -InputObject <object>` | Wandelt rekursiv `PSCustomObject` (aus `ConvertFrom-Json`) in verschachtelte `Hashtable`s um - Ersatz für `ConvertFrom-Json -AsHashtable`, das erst ab PowerShell 6 existiert. Arrays bleiben Arrays, auch mit nur einem oder keinem Element. |
| `Get-DocFlowRelativePath -BasePath <string> -FullPath <string>` | Berechnet den relativen Pfad rein per String-Verarbeitung (bewusst **kein** `[Uri]::MakeRelativeUri` - das erkennt POSIX-Pfade ohne `file://`-Schema nicht zuverlässig als absolut). Grundlage für den rechnerunabhängigen Dedup-Key in `Copy-NewFiles`. |
| `Copy-DocFlowFileWithRetry -Source <string> -Destination <string> [-RetryCount <int> = 2] [-RetryDelaySeconds <int> = 5]` | Kopiert (mit `-LiteralPath`, damit `[ ]` im Dateinamen nicht als Wildcard gilt) mit bis zu `RetryCount` Wiederholungen und prüft nach jedem Versuch, ob die Zieldatei existiert und gleich groß ist. Gibt `$true`/`$false` zurück; loggt fehlgeschlagene Einzelversuche als `Warning`, das endgültige Scheitern als `Error`. |
| `Test-DocFlowFileReady -File <FileInfo> [-MinAgeMinutes <double>]` | `$false`, wenn die Datei jünger als `MinAgeMinutes` ist oder sich nicht exklusiv öffnen lässt (wird noch geschrieben/synchronisiert). |
| `Write-DocFlowHintFile -Path <string> -Content <string> -LogMessage <string>` | Schreibt eine Hinweisdatei, falls sie noch nicht existiert. Fehler beim Schreiben werden als `Error` geloggt statt den Lauf abzubrechen. |
| `Remove-DocFlowHintFile -Path <string> -Reason <string>` | Löscht eine Hinweisdatei (falls vorhanden) und loggt den Grund. Im DryRun nur Ankündigung. |
| `Remove-DocFlowOrphanedHintFiles -Directory <string> -HintSuffixes <string[]> [-Recurse <bool>]` | Löscht Hinweisdateien mit einer der Endungen, deren zugehörige Datei nicht mehr existiert. |

## `Config.ps1` - Konfiguration laden

| Funktion | Zweck |
|---|---|
| `Import-DocFlowConfig -Path <string>` (Alias `Load-Config`) | Lädt die `.psd1`-Konfiguration, validiert Pflichtfelder (`sources`, `namingConventions`, sowie `aufgabenRoot` oder `targets`), setzt Defaults für alle optionalen Felder, ergänzt die Hinweisdatei-Suffixe in `sources[].excludePatterns` und ersetzt zum Schluss die `{name}`-Verweise auf den `paths`-Block. Details siehe [Configuration.md](konfiguration.md). |
| `Get-DocFlowConfigSection -Config <hashtable> -Name <string>` | Liefert einen Abschnitt der Konfiguration (z. B. `duplicateHint`) und legt ihn als leere `Hashtable` an, falls er fehlt. Nicht exportiert. |
| `Set-DocFlowDefaults -Section <hashtable> [-IfMissing <hashtable>] [-IfEmpty <hashtable>]` | Setzt Standardwerte in einem Abschnitt. `-IfMissing` füllt nur fehlende Schlüssel (ein ausdrückliches `$false` oder `0` bleibt erhalten, z. B. `enabled`, `minFileAgeMinutes`, `copyRetry.count`). `-IfEmpty` ersetzt auch leere Werte (z. B. Suffixe, Meldungen, `stateFile`). Nicht exportiert. |
| `Resolve-DocFlowPathVariables -Paths <hashtable>` | Löst die Einträge des `paths`-Blocks auf, inklusive Verweisen untereinander. Wirft bei unbekannten Namen und zirkulären Verweisen. |
| `Expand-DocFlowPathValue -Value <string> -PathVariables <hashtable> [-FieldName <string>]` | Ersetzt alle `{name}`-Verweise in einem einzelnen Pfadwert. `FieldName` erscheint in der Fehlermeldung, wenn ein Name fehlt. |
| `Expand-DocFlowConfigPaths -Config <hashtable>` | Wendet `Expand-DocFlowPathValue` auf alle Pfadfelder der Konfiguration an (Liste siehe [Configuration.md](konfiguration.md#paths-optional-empfohlen---pfade-nur-ganz-oben-anpassen)). |

## `State.ps1` - Zustandsdatei

| Funktion | Zweck |
|---|---|
| `Import-DocFlowState -StatePath <string>` (Alias `Load-State`) | Lädt `.docflow-state.json`; liefert `@{ processed = @{}; reviewedFiles = @{} }`, falls die Datei fehlt oder nicht lesbar ist. Listen wie `targets` bleiben auch mit einem oder keinem Element Listen. |
| `Save-State -StatePath <string> -State <hashtable>` | Schreibt den State als JSON zurück, legt das Zielverzeichnis bei Bedarf an. |
| `Remove-DocFlowExpiredState -State <hashtable> -RetentionYears <int>` | Entfernt aus `processed` und `reviewedFiles` alle Einträge, deren `processedAt` älter als `RetentionYears` ist (Einträge ohne auswertbaren Zeitstempel bleiben erhalten). Gibt die Anzahl entfernter Einträge zurück. Wird in `Invoke-DocFlowEngine` vor `Save-State` aufgerufen (siehe [Configuration.md](konfiguration.md), `stateRetentionYears`). |

## `Lock.ps1` - Multi-Machine-Lock

| Funktion | Zweck |
|---|---|
| `Test-DocFlowLockFresh -LockPath <string> -TimeoutMinutes <int>` | Prüft, ob eine vorhandene Lock-Datei jünger als `TimeoutMinutes` ist. |
| `Lock-DocFlowRun -LockPath <string> [-TimeoutMinutes <int> = 15]` | Versucht die Lock-Datei zu erwerben (Inhalt: `Hostname\|PID\|Zeitstempel`, ermittelt über `[System.Environment]::MachineName` - plattformunabhängig, PS-5.1-kompatibel). Gibt `$false` zurück, wenn eine frische Lock-Datei existiert. |
| `Unlock-DocFlowRun -LockPath <string>` | Entfernt die Lock-Datei. |

## `Naming.ps1` - Namenskonvention, Umbenennung, Whitelist

| Funktion | Zweck |
|---|---|
| `Get-TargetFileName -File <FileInfo> -Rules <array> -DefaultFormat <string>` | Ermittelt den Zieldateinamen anhand der ersten passenden `namingConventions`-Regel (per `[regex]::Match`, nicht `-match`/`$Matches` - damit liefert eine nicht getroffene optionale Gruppe wie `versiontag` garantiert einen leeren String statt eines fehlenden Platzhalters). Fällt sonst auf `DefaultFormat` zurück. |
| `Get-FileCategory` / `Resolve-CategoryTarget` | Legacy-Routing über führende Buchstaben + `categoryRoutes`; in der aktuellen Konfiguration ungenutzt. |
| `Get-FilePraefixSuffix -File <FileInfo> -Rules <array>` | Extrahiert `praefix`/`suffix` aus dem ersten Regel-Treffer mit beiden benannten Gruppen. |
| `Get-FileInitialsFromName -Name <string> -Rules <array>` | Extrahiert die `initials`-Gruppe (Schüler-Kürzel) aus einem Dateinamen (ohne Endung) anhand des ersten Regel-Treffers mit dieser Gruppe (per `[regex]::Match`, nicht `-match`/`$Matches`, analog zu `Get-TargetFileName`). Kernfunktion, von `Get-FileInitials` (Vorwärtslauf) und `Copy-ReviewedFiles` (Rücklauf, auf den vom `_k-...`-Marker befreiten Namen angewendet) gemeinsam genutzt. |
| `Get-FileInitials -File <FileInfo> -Rules <array>` | Dünner Wrapper um `Get-FileInitialsFromName` für den `FileInfo`-Fall. |
| `Test-DocFlowPraefixSuffixKnown -Registry <hashtable> -Praefix <string> -Suffix <string>` | Prüft, ob die **Kombination** in `Registry.Paare` steht. Fehlt `Paare` (selbst gebaute Registry), werden Präfix und Suffix wie früher einzeln geprüft. |
| `Get-DocFlowRegistryLines -Path <string>` | Liest eine Whitelist-Datei zeilenweise (getrimmt) und trennt dabei versehentlich zusammengeklebte Einträge wie `suffix=Registrysuffix=Arrays` wieder in einzelne Zeilen auf. Gemeinsame Lesebasis von `Get-PraefixSuffixRegistry` und `Sync-ProjectRoutesFromSeed`. Nicht exportiert. |
| `ConvertTo-DocFlowRegistryStructure -Lines <string[]>` / `ConvertFrom-DocFlowRegistryStructure -Structure <object>` | Wandelt Whitelist-Zeilen in eine nach Präfix gruppierte Struktur (`Groups`, `Orphans`, `Invalid`) und zurück in Zeilen. Basis für Laden und Abgleich. Nicht exportiert. |
| `Get-PraefixSuffixRegistry -Path <string>` | Lädt `project-routes.txt`: `Paare` (Präfix → `HashSet` erlaubter Suffixe) sowie zur Abwärtskompatibilität `Praefixe` und `Suffixe` (alle case-insensitive). Protokolliert Pfad und Anzahl der Kombinationen (`Info`), die vollständige Liste (`Debug`) sowie ungültige Zeilen und Suffixe vor dem ersten Präfix (`Warning`). |
| `Sync-ProjectRoutesFromSeed -SeedPath <string> -TargetPath <string>` | Legt `TargetPath` aus der Seed-Datei an, falls er fehlt. Sonst werden beide nach Präfix gruppiert zusammengeführt (fehlende Kombinationen aus der Seed-Datei unter den passenden Präfix, eigene Kombinationen bleiben) und die Datei nur bei einer Änderung neu geschrieben (UTF-8). No-op, falls beide Pfade identisch sind oder die Seed-Datei fehlt. |
| `Write-NamingConventionHint -File <FileInfo> -HintConfig <hashtable> [-PraefixSuffix <PSCustomObject>]` | Erzeugt über `Write-DocFlowHintFile` die individuelle Hinweisdatei `<Dateiname><fileNameSuffix>` neben der betroffenen Datei (keine Wirkung, falls die Datei schon existiert). `-PraefixSuffix` optional, damit die Meldung `{praefix}`/`{suffix}` referenzieren kann. |
| `Get-DocFlowSubmissionParts -Name <string> -Rules <array>` | Zerlegt einen Dateinamen (ohne Endung) in `Initials`, `Version` (ohne `_v` = 1), `Praefix`, `Suffix`, `Aufgabennummer`. |
| `Find-DocFlowExistingSubmission -File <FileInfo> -TargetDirectory <string>` | Sucht im Zielordner eine Datei `<Datum>_<gleicher Dateiname>`. |
| `New-DocFlowSubmissionStem -Initials <string> [-Version <int> = 0] -Praefix <string> -Suffix <string> -Aufgabennummer <string>` | Setzt den Dateinamen-Stamm `initialen[_vN]_praefix_suffix_aufgabennummer` zusammen (ohne `_vN` bei Version 0). Einzige Stelle für dieses Schema, genutzt von `Get-DocFlowSuggestedVersionName`, `Get-DocFlowNameProposal` und `Rename-ExistingAufgaben.ps1`. |
| `Get-DocFlowSuggestedVersionName -File <FileInfo> -Rules <array> [-TargetDirectory <string>] [-ReviewMarkerPattern <string> = '_k-[A-Za-z0-9]+$']` | Schlägt den Dateinamen mit der nächsten freien Versionsnummer vor (höchste Version derselben Aufgabe im Schüler- und Zielordner + 1). Korrigierte Versionen werden mitgezählt; `Copy-NewFiles` übergibt dafür das konfigurierte `reviewMarker.pattern`. |
| `Write-DocFlowDuplicateHint -File <FileInfo> -HintConfig <hashtable> [-SuggestedName <string>]` | Schreibt den `.BEREITS-ABGEGEBEN.txt`-Hinweis (`duplicateHint`). |
| `Get-DocFlowNameProposal -Name <string> -Rules <array> [-Registry <hashtable>] [-KnownInitials <string>] [-Placeholder <string> = 'PLATZHALTER']` | Liefert den Namensvorschlag als Objekt (`Stem`, `Compliant`, `Initials`, `Version`, `Praefix`, `Suffix`, `Aufgabennummer`, `HasPlaceholder`, `Notes`). Die Teilschritte liegen in den nicht exportierten Funktionen `Get-DocFlowNameTokens` (Umlaute ersetzen, Kopie-Zusätze entfernen, in Tokens zerlegen), `Find-DocFlowTokenIndex`, `Find-DocFlowPraefixSuffixInTokens` (Registry mit Paaren, flache Registry oder keine) und `Find-DocFlowRegistryPairInTokens` (Paar-Erkennung: beide Tokens, zusammengeschrieben, nur Suffix, nur Präfix, mehrdeutig). |
| `Get-DocFlowMigratedFileName -Name <string> -Rules <array> [-Registry <hashtable>] [-Placeholder <string> = 'PLATZHALTER']` | Liefert nur `Stem` aus `Get-DocFlowNameProposal`. Best-effort-Ableitung eines regelkonformen Namens aus einem beliebigen Alt-Dateinamen: unverändert, falls `Name` bereits einer `Rules`-Regel entspricht; sonst Token-Suche nach genau einem bekannten Präfix/Suffix aus `Registry` (`Get-PraefixSuffixRegistry`), führende 2-5 Buchstaben als Initialen, letztes verbleibendes Token als Aufgabennummer - für jeden nicht bestimmbaren Teil `Placeholder`. Kernlogik von `scripts/Rename-ExistingAufgaben.ps1` (siehe unten). |

## `CopyForward.ps1` - Quell-Scan und Vorwärtskopie

| Funktion | Zweck |
|---|---|
| `Get-SourceFiles -Source <hashtable> -ResolvedPath <string>` | Scannt einen aufgelösten Quellordner nach `includePatterns`, wendet `excludePatterns` an. |
| `Initialize-DocFlowTargetDirectories -Targets <array>` (Alias `Ensure-TargetDirectories`) | Legt fehlende Zielordner an (sofern `createIfMissing` nicht `$false` ist) und normalisiert `target.path` auf den aufgelösten Pfad. |
| `Test-DocFlowInsideNamedFolder -DirectoryName <string> -FolderName <string>` | Prüft, ob ein Pfad-Segment exakt `FolderName` entspricht (z. B. `Korrigiert`) - robuster als ein `-like`-Muster auf den Dateinamen, da er auf ganzen Pfadsegmenten statt nur dem Dateinamen prüft. |
| `Get-DocFlowFileFingerprint -File <FileInfo>` | Liefert `Length` und `LastWriteUtcSeconds` einer Datei für den Zustandseintrag. |
| `Test-DocFlowSameFileVersion -Entry <hashtable> -File <FileInfo> [-ToleranceSeconds <int> = 2]` | Vergleicht einen Zustandseintrag mit der aktuellen Datei (gleiche Größe, Änderungszeit innerhalb der Toleranz). |
| `Test-DocFlowHintFileName -Name <string> [-HintSuffixes <string[]>]` | `$true`, wenn der Name auf eine der Hinweis-Endungen endet. Nicht exportiert. |
| `Test-DocFlowKnownFileUnchanged -Entry <hashtable> -File <FileInfo>` | `$true`, wenn ein bereits verarbeiteter Eintrag zur aktuellen Datei passt. Trägt bei alten Einträgen ohne Größe/Zeit beides nach und gilt dann als unverändert. Nicht exportiert. |
| `Resolve-DocFlowAufgabenRoute -File <FileInfo> [-PraefixSuffix ...] -AufgabenRoot <string> -Rules <array> [-PraefixSuffixRegistry ...] [-RegistryFilePath ...] [-NamingConventionHint ...] [-KuerzelRoutes ...] [-KuerzelRoutesFilePath ...]` | Routing nach `aufgabenRoot`: liefert `TargetPath` (`<aufgabenRoot>/<Präfix>/<Suffix>`, registriert dabei das Kürzel) oder `Rejected` (Warnung plus Hinweisdatei, falls `namingConventionHint` aktiv). Ist der Hinweis abgeschaltet, geht die Datei an das übrige Routing weiter. Nicht exportiert. |
| `Get-DocFlowWhitelistRejectionReason -PraefixSuffix <PSCustomObject> [-Registry ...]` | Text für die Whitelist-Warnung: unbekanntes Präfix, Suffix nicht für dieses Präfix erlaubt, oder (ohne Paare) unbekanntes Präfix und/oder Suffix. Nicht exportiert. |
| `Copy-DocFlowFileToTargets -File <FileInfo> -Targets <array> -TargetFileName <string> -SourceRoot <string> [-RetryCount <int>] [-RetryDelaySeconds <int>]` | Kopiert in alle Ziele (mit `preserveSubfolders`, Ordneranlage, Ziel-Existenz-Check, DryRun). Liefert `Succeeded` und `TargetPaths`. Nicht exportiert. |
| `Complete-DocFlowProcessedFile -State <hashtable> -SourceKey <string> -File <FileInfo> [-TargetPaths <array>] [-HintSuffixes <array>]` | Schreibt den Zustandseintrag und entfernt die Hinweisdateien zur Datei. Nicht exportiert. |
| `Write-DocFlowResubmissionHint -File <FileInfo> -Rules <array> [-DuplicateHint <hashtable>] [-TargetDirectory <string>] [-ReviewMarkerPattern <string>]` | Schreibt den `duplicateHint` mit Versionsvorschlag, falls aktiviert. Nicht exportiert. |
| `New-DocFlowRunContext -Config <hashtable>` | Baut aus der geladenen Konfiguration den Laufkontext: `State` und `StatePath` (lädt die Zustandsdatei), aufgelöstes `AufgabenRoot`, `PraefixSuffixRegistry` und `RegistryFilePath` (gleicht vorher mit `projectRoutesSeedFile` ab), `KuerzelRoutes` und `KuerzelRoutesFilePath` sowie `ExcludePaths` (alle `targets[].path` und `aufgabenRoot`). Ohne `projectRoutesFile` ist die Registry leer (`Praefixe`/`Suffixe` ohne `Paare`). |
| `New-DocFlowRunContextObject -State <hashtable> [-StatePath ...] [-AufgabenRoot ...] [-PraefixSuffixRegistry ...] [-RegistryFilePath ...] [-KuerzelRoutes ...] [-KuerzelRoutesFilePath ...] [-ExcludePaths ...]` | Erzeugt das Kontext-Objekt aus Einzelwerten, ohne etwas zu laden. Gemeinsame Basis von `New-DocFlowRunContext` und `Copy-NewFiles`. Nicht exportiert. |
| `Invoke-DocFlowCopyForward -Config <hashtable> -Context <PSCustomObject>` | Kernfunktion des Vorwärtslaufs. Liest die Einstellungen aus der Konfiguration (`sources`, `targets`, `namingConventions`, `defaultNameFormat`, `categoryRoutes`, `namingConventionHint`, `duplicateHint`, `minFileAgeMinutes`, `copyRetry`, `reviewMarker.korrigiertFolderName`/`pattern`), den Laufzustand aus dem Kontext. Erwartet eine Konfiguration, wie `Import-DocFlowConfig` sie liefert (alle Defaults gesetzt). Reihenfolge je Datei: Ausschlüsse → Zustand (unverändert: überspringen; ohne Größe/Zeit: nachtragen; überschrieben: `duplicateHint`) → `Test-DocFlowFileReady` → Routing nach `aufgabenRoot`/Präfix-Suffix-Paar/`categoryRoutes`/`targets` → gleichnamige Abgabe im Zielordner (identisch: übernehmen, sonst `duplicateHint`) → `Copy-DocFlowFileWithRetry` → Zustandseintrag nur bei Erfolg, danach Hinweisdateien zur Datei entfernen. Vorab werden verwaiste Hinweisdateien im Quellordner entfernt. |
| `Copy-NewFiles -Sources ... -Targets ... -State ... -Rules ... -DefaultNameFormat ... [-AufgabenRoot ...] [-PraefixSuffixRegistry ...] [-RegistryFilePath ...] [-KuerzelRoutes ...] [-KuerzelRoutesFilePath ...] [-KorrigiertFolderName ...] [-ReviewMarkerPattern ...] [-NamingConventionHint ...] [-DuplicateHint ...] [-MinFileAgeMinutes ...] [-RetryCount ...] [-RetryDelaySeconds ...] [-ExcludePaths ...]` | Kompatibilitätshülle mit der bisherigen Signatur. Baut aus den Parametern Konfiguration und Kontext und ruft `Invoke-DocFlowCopyForward` auf; das Verhalten ist dasselbe. Neue Aufrufe sollten `Invoke-DocFlowCopyForward` verwenden. |

## `CopyBack.ps1` - "Korrigiert"-Rücklauf

| Funktion | Zweck |
|---|---|
| `Get-KuerzelRoutes -Path <string>` | Lädt `kuerzel-routes.txt` in eine `Hashtable` (Kürzel → Schülerordner-Pfad, case-insensitive). |
| `Register-Kuerzel -Routes <hashtable> [-RoutesFilePath <string>] -Kuerzel <string> -SourcePath <string>` | Trägt ein neues Kürzel einmalig ein (in-memory + Datei), no-op falls schon bekannt. Wird aus `Copy-NewFiles` heraus mit `$item.DirectoryName` aufgerufen (nicht der Quellwurzel - wichtig bei einem gemeinsamen, rekursiv gescannten Austauschordner mit Schüler-Unterordnern). |
| `Copy-ReviewedFiles -AufgabenRoot <string> -ReviewMarker <hashtable> -KuerzelRoutes <hashtable> -State <hashtable> -Rules <array> [-UnknownKuerzelHint <hashtable>] [-RetryCount <int>] [-RetryDelaySeconds <int>]` | Marker-Erkennung ohne Rücksicht auf Groß-/Kleinschreibung; Kopie über `Copy-DocFlowFileWithRetry`, Zustandseintrag nur bei Erfolg; entfernt danach einen vorhandenen `.KUERZEL-UNBEKANNT.txt`-Hinweis sowie vorab alle verwaisten. | Durchsucht `aufgabenRoot` rekursiv nach `_k-...`-markierten Dateien. Der Marker-Text selbst wird nirgends ausgewertet - stattdessen entfernt die Funktion den Marker sowie ein führendes `{date}_` und ermittelt über `Get-FileInitialsFromName -Rules $Rules` das Schüler-Kürzel aus dem verbleibenden (unveränderten) Dateinamen. Schlägt dieses Kürzel in `KuerzelRoutes` nach, kopiert nach `<Schülerordner>/<Korrigiert>/`. Überspringt bereits zurückkopierte Dateien (State) und Dateien, die bereits im `Korrigiert`-Ordner selbst liegen. |

## `Rename.ps1` - Hilfsfunktionen für `Rename-ExistingAufgaben.ps1`

| Funktion | Zweck |
|---|---|
| `ConvertTo-DocFlowKuerzelByFolder -KuerzelRoutes <hashtable>` | Kehrt die Kürzel-Registry (Kürzel → Schülerordner) zu Ordner → Kürzel um. Abschließende `\`/`/` werden entfernt; zeigen mehrere Kürzel auf denselben Ordner, gilt das erste. |
| `Get-DocFlowFolderInitials -Files <FileInfo[]> -Rules <array> [-Registry ...] [-KuerzelByFolder <hashtable>] [-Placeholder <string> = 'PLATZHALTER']` | Ermittelt pro Ordner die Initialen für Umbenennungen. Liefert eine `Hashtable` Ordner → Objekt mit `Initials` und `Source`. Reihenfolge: die häufigsten Initialen regelkonformer Dateien im Ordner (`aus regelkonformen Dateien im Ordner`), sonst der Eintrag aus der Kürzel-Registry (`aus der Kürzel-Registry`), sonst eindeutig erratene Initialen aus Dateien mit erkanntem Präfix oder Suffix (`aus anderen Dateinamen im Ordner`). Sind die erratenen Initialen mehrdeutig, fehlt der Ordner im Ergebnis. |
| `Undo-DocFlowRename -ReportPath <string>` | Macht die Zeilen eines `-Apply`-Berichts mit `Ausgefuehrt = ja` in umgekehrter Reihenfolge rückgängig. Überspringt mit Warnung Dateien, die nicht mehr existieren, und alte Namen, die inzwischen wieder belegt sind. Liefert `Restored` und `Total`. Wirft, wenn der Bericht fehlt. |

## Aufrufreihenfolge in `Invoke-DocFlowEngine`

```
Import-DocFlowConfig
  → Lock-DocFlowRun (falls lockFile konfiguriert und nicht -DryRun)
    → Initialize-DocFlowTargetDirectories
    → New-DocFlowRunContext
        → Import-DocFlowState
        → Sync-ProjectRoutesFromSeed (falls projectRoutesFile + projectRoutesSeedFile konfiguriert)
        → Get-PraefixSuffixRegistry (falls projectRoutesFile konfiguriert)
        → Get-KuerzelRoutes (falls reviewMarker.kuerzelRoutesFile konfiguriert)
    → Invoke-DocFlowCopyForward (Vorwärtslauf, inkl. Register-Kuerzel bei Bedarf)
    → Copy-ReviewedFiles      (Rücklauf, nur falls aufgabenRoot + reviewMarker.enabled)
    → Remove-DocFlowExpiredState (falls stateRetentionYears > 0)
    → Save-State              (nicht im DryRun)
  → Unlock-DocFlowRun (finally-Block, immer falls Lock erworben wurde)
```

## Eigenständige Skripte (kein Teil von `Invoke-DocFlowEngine`)

Diese beiden Skripte importieren zwar dasselbe Modul, laufen aber unabhängig vom regulären
`Invoke-DocFlowEngine`-Ablauf und werden nicht über die Aufgabenplanung mit ausgeführt (außer
explizit so eingerichtet, siehe README).

| Skript | Zweck |
|---|---|
| `scripts/Invoke-OneDriveSync.ps1` | Kein Modul-Import. Beendet den OneDrive-Prozess des Benutzers und startet ihn neu (`Stop-Process`/`Start-Process`), um einen frischen Sync-Check anzustoßen - kein offizieller, garantierter "Sync jetzt"-Befehl existiert. `-ExecutablePath` überschreibbar; ohne gefundene `OneDrive.exe` (z. B. `$env:LOCALAPPDATA` nicht gesetzt, wie auf macOS/Linux) bricht das Skript kontrolliert mit einer Warnung ab statt mit einem Fehler. |
| `scripts/Rename-ExistingAufgaben.ps1 [-Path <string[]>] [-ConfigPath <string>] [-Apply] [-ReportPath <string>] [-Undo <string>] [-MarkAsProcessed] [-IncludeUnrecognized] [-Placeholder <string>]` | Manuell auszuführendes Migrationsskript für Alt-Bestand. Ohne `-Path` werden alle `sources` durchsucht. Nutzt `Import-DocFlowConfig`, `Get-PraefixSuffixRegistry`, `ConvertTo-DocFlowKuerzelByFolder`, `Get-DocFlowFolderInitials`, `Get-DocFlowNameProposal`, `New-DocFlowSubmissionStem`, `Lock-DocFlowRun`, `Import-DocFlowState`, `Save-State` und für `-Undo` `Undo-DocFlowRename`. Ohne `-Apply` reine Vorschau (keine Änderung), der Bericht landet als CSV unter `-ReportPath`. Mit `-Apply`: benennt Dateien um; ist der Zielname schon vergeben, wird die Versionsangabe hochgezählt (`_v2`, `_v3`, …). Mit `-MarkAsProcessed` werden umbenannte Dateien unter demselben Dedup-Key-Format wie im Vorwärtslauf (`"$($source.path)\|<relativer Pfad>"`) in `.docflow-state.json` eingetragen, damit `Invoke-DocFlowEngine` sie beim nächsten Lauf nicht erneut als neue Datei behandelt. `-Undo <Bericht>` macht einen `-Apply`-Lauf rückgängig. |


---

## Fehlersuche

Jede Funktion schreibt ihre Meldungen über `Write-Log` (Level `Trace` bis `Error`). Die Zuordnung von Meldung zu Teilmodul steht in [Debugging und Fehlersuche](debugging.md#fehlermeldungen). Einzelne Funktionen lassen sich im Debugger prüfen: Die Teildateien werden per Dot-Sourcing geladen, ein Haltepunkt mit `Set-PSBreakpoint -Script .\scripts\DocFlowEngine\<Teilmodul>.ps1 -Line <n>` funktioniert nach `Import-Module … -Force`.
