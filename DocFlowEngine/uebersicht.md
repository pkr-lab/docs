# DocFlowEngine – Dokumentation

Diese Detail-Dokumentation ergänzt die Kurzanleitung im [Haupt-README](index.md).
Sie ist nach Themen aufgeteilt:

- [Compatibility-und-Admin-Rechte.md](compatibility-und-admin-rechte.md) - Nachweis: läuft alles mit reinem PowerShell 5.1 und ohne Admin-Rechte?
- [Configuration.md](konfiguration.md) - Alle Optionen von `config/docflow-config.psd1` im Detail
- [Module-Reference.md](module-reference.md) - Funktionsreferenz der Teilmodule unter `scripts/DocFlowEngine/`
- [Implementation-Notes.md](implementation-notes.md) - Begründungen für nicht-offensichtliche Implementierungsentscheidungen im Code
- [Logging.md](logging.md) - Log-Level, Log-Format, Verhalten im DryRun
- [Debugging.md](debugging.md) - Fehlermeldungen, Fehlersuche, lokales Debugging
- [Testing.md](testing.md) - Pester-Tests installieren und ausführen

Weiterführende Konzeptdokumente (in Ordner [konzepte/](konzepte/multi-machine-setup.md), getrennt von der übrigen
Detail-Dokumentation, da sie eigenständige Entscheidungsdokumente sind):

- [Konzepte/MULTI-MACHINE-SETUP.md](konzepte/multi-machine-setup.md) - Betrieb auf mehreren Rechnern
- [Konzepte/ERWEITERUNGSKONZEPT.md](konzepte/erweiterungskonzept.md) - Herleitung der Pro-Datei-Hinweise, des "Korrigiert"-Rücklaufs, der Präfix/Suffix-Whitelist und der Modularisierung
