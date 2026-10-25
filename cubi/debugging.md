# Debugging und Fehlersuche

Zentrale Seite für cubi. Fehler, die nur einen Bereich betreffen, stehen auf der Seite des Bereichs ([ArgoCD](argocd.md), [ArgoCD-Features](argocd-features.md), [Connect](connect.md), [Tailscale](tailscale.md), [GitHub](github.md), [Kubernetes](kubernetes.md), [Helm](helm.md), [Tokens](tokens.md), [Login](login.md)). Das projektübergreifende Kapitel ist die [zentrale Fehlersuche](../capulus/capulus-core/2-betrieb-hardware/20070-troubleshooting-zentral.md).

---

## Wie cubi Fehler behandelt

- cubi schreibt **kein Log** und kennt weder Log-Level noch einen Debug-Schalter. Es gibt keine Umgebungsvariable, die mehr Ausgabe erzeugt.
- Fehler, die cubi erwartet, sind `CubiError`. `ui.guard()` zeigt sie als roten Kasten **Fehler** und führt zum Menü zurück. cubi läuft weiter.
- Ladefehler in Tabellen erscheinen in der Tabelle als rote Zeile `✖ <Meldung>`. Die Tasten (`r`, Hotkeys, Esc) funktionieren weiter, `r` versucht es erneut.
- Fehlermeldungen von `kubectl`, `argocd`, `gh`, `helm` und `tailscale` gibt cubi mit der ersten nützlichen Zeile von stderr wieder (`shell.first_line`). Klebt eine JSON-Logzeile mit `msg` davor, nimmt cubi deren `msg`. Zeilen im Stil `E0102 12:34:56` (Kubernetes-Logformat) werden als Rauschen übersprungen.
- Alle Unterprozesse haben ein Timeout (5 bis 90 Sekunden je Aufruf). Danach folgt `Zeitüberschreitung bei: <befehl>`.
- Alles andere (Programmierfehler, unerwartete Datenformate) endet mit einem Python-Traceback. Strg+C beendet ohne Traceback mit Exit-Code 130, der Terminal-Modus wird dabei wiederhergestellt.

---

## Fehlermeldungen von cubi

### Start und Konfiguration

| Meldung | Ursache | Lösung |
|---|---|---|
| `cubi benötigt ein interaktives Terminal` (Exit 1) | `stdin` oder `stdout` ist kein TTY (Pipe, Cron, IDE-Konsole ohne TTY) | in einem echten Terminal starten |
| `Unbekannter Bereich: <name>` (Exit 2) | Tippfehler oder Alias fehlt | `cubi --help` |
| `Keine targets.yaml gefunden` | keine Konfigurationsdatei | siehe [Konfiguration](konfiguration.md) |
| `Ungültige Config <pfad>: …` | YAML-Fehler oder oberste Ebene kein Mapping | Datei korrigieren |
| `Keine Einträge vorhanden` | eine Auswahl wurde ohne Einträge geöffnet (zum Beispiel leere Namespace-Liste) | Quelle prüfen (Kontext, Config) |

### Unterprozesse

| Meldung | Ursache | Lösung |
|---|---|---|
| `<werkzeug> ist nicht installiert oder nicht im PATH` | `kubectl`, `argocd`, `gh`, `tailscale`, `helm`, `ssh` oder `python3` fehlt | Werkzeug installieren, `which <werkzeug>` |
| `Zeitüberschreitung bei: <befehl>` | Befehl brauchte länger als das Timeout (siehe unten) | Verbindung zum Cluster, zu GitHub oder zu Tailscale prüfen |
| `Befehl fehlgeschlagen: <befehl>` | Rückgabecode ≠ 0 ohne Text auf stderr | Befehl direkt ausführen |
| `Ungültige Ausgabe von: <befehl>` | erwartetes JSON kam nicht (Warnung oder Fehlertext in stdout) | Befehl direkt ausführen, Werkzeug aktualisieren |
| `Beendet mit Code N` | ein von cubi gestarteter Befehl (Login, Sync, Skript) endete mit Fehler | der Befehl steht darüber (`$ …`), selbst ausführen |

Timeouts: Standard 30 Sekunden, `ssh -G` 5 Sekunden, Statusabfragen für Login und Tailscale 15 Sekunden, einzelne `kubectl`-Abfragen für Namespace und Kontext 10 Sekunden, `gh`-Hilfsabfragen 20 Sekunden, Ausgaben wie `argocd app diff` oder `helm status` 90 Sekunden. Befehle mit Terminal-Übergabe (`ssh`, `kubectl logs -f`, `kubectl exec`) haben kein Timeout.

### Bedienung

| Meldung | Ursache | Lösung |
|---|---|---|
| `Mindestens 1 Eintrag wählen` | Enter in einer Mehrfachauswahl ohne Häkchen | mit Space wählen |
| Anzeige springt oder bleibt hängen nach einem Absturz | Terminal blieb im Rohmodus (kein Echo) | `reset` oder `stty sane` eingeben |
| Tabelle wird abgeschnitten | Zeilen werden auf die Terminalbreite gekürzt, nie umgebrochen | Terminal verbreitern |
| Tasten reagieren nicht, `unknown` | unbekannte Escape-Sequenz | Esc, dann Pfeiltasten; andere Terminal-Emulation (`TERM`) testen |

---

## Checkliste „cubi funktioniert nicht“

1. **Startet es?** `cubi --help`. Kommt ein `ModuleNotFoundError`, fehlen `rich` oder `pyyaml` ([Installation](installation.md)).
2. **Echtes Terminal?** Ohne TTY startet cubi nicht.
3. **Konfiguration:** Welche Datei nutzt cubi? Reihenfolge `CUBI_CONFIG`, `~/.config/cubi/targets.yaml`, `config/targets.yaml`. Eine vergessene Datei in `~/.config/cubi/` überdeckt das Repo.
4. **Werkzeug in der Shell:** den Befehl hinter der Ansicht selbst ausführen (`kubectl get pods -A`, `gh run list`, `tailscale status`). Klappt er dort nicht, ist es kein cubi-Problem.
5. **Logins:** `cubi login`: alle drei Zeilen grün?
6. **Kontext:** `kubectl config current-context`, `argocd context`.
7. **Netz:** VPN oder Tailscale verbunden ([Tailscale](tailscale.md))?
8. **Rechte:** `kubectl auth can-i patch applications.argoproj.io -n argocd` bei Änderungsfunktionen.

---

## Typische Probleme zwischen den Repos

| Symptom | Ursache | Lösung |
|---|---|---|
| `cubi tokens` findet das Register nicht | `tokens.repo` zeigt nicht auf [capulus-core](../capulus/capulus-core/index.md) | Pfad in der Config setzen |
| `cubi tokens`: Runbook-Link öffnet nichts | Runbook mit Präfix `pkr-lab-docs:` und `tokens.docs_repo` falsch | Pfad zum Checkout dieses Repos setzen |
| Rotieren bricht mit `Im Register fehlt rotation.command` ab | `scripted`-Einträge im Register ohne `command` | siehe [Tokens](tokens.md#bekannte-einschränkung-scripted-ohne-rotationcommand) |
| Branch aus *Feature-Testing* wird nach Minuten wieder zurückgesetzt | App gehört einem ApplicationSet oder einer App-of-Apps, die [Root-Applicationset](../capulus/capulus-core/6-hintergruende/60030-argocd-und-bootstrap.md) setzt den Wert zurück | Warnung der Nachkontrolle beachten; dauerhaft in der Quelle ändern |
| Auto-Sync ist nach einem Neu-Einspielen der ApplicationSets wieder an | die von cubi gesetzten Regeln `ignoreApplicationDifferences` stehen nur im Cluster | nach dem Test erneut ausführen oder dauerhaft im Git ändern |
| Pods einer App kommen nach *Pods skalieren* wieder hoch | Auto-Sync war an und wurde nicht abgeschaltet (Eingriff von außen) oder ein HPA greift | [HPA](../capulus/capulus-core/b-kubernetes-gitops/b0040-hpa-autoscaling.md), Nachkontrolle lesen |
| `cubi connect` kennt einen neuen Host nicht | Host fehlt in `targets.yaml` oder in `~/.ssh/config` | beides ergänzen, siehe [Konfiguration](konfiguration.md); Hosts selbst: [capulus-ansible](../capulus/capulus-ansible/index.md) |

---

## Lokal debuggen

Siehe auch [Entwicklung](entwicklung.md).

```bash
cd ~/gitlab/ci/cubi
python3 -m unittest discover -s tests          # alle Tests (201 Tests, rund 17 Sekunden)
(cd tests && python3 -m unittest test_scale -v) # einzelnes Testmodul; im Ordner tests/ ausführen,
                                               # weil die Tests `import helpers` ohne Paketpfad nutzen
python3 -X dev bin/cubi argocd                 # Development-Modus für Warnungen
python3 -m pdb bin/cubi tokens                 # Debugger; Haltepunkt mit breakpoint() im Modul
```

Hilfreich:

- Unterprozess-Aufrufe findet man in `shell.py` (`capture`, `run`, `replace`). Ein `print(cmd)` dort zeigt alle Befehle, die cubi ausführt.
- Die Modul-`run()`-Funktionen sind Einstiege. `ui.guard(<funktion>, …)` kapselt Fehler. Für die Fehlersuche kann man `guard` im Debugger umgehen, um den Traceback zu sehen.
- Reine Planlogik (`plan`, `plan_set`, `normalize`) ist ohne Cluster in den Tests aufgerufen, siehe `tests/test_autosync.py`, `tests/test_feature_testing.py`, `tests/test_scale.py`.
- Terminalverhalten testen die Dateien `test_inline_pty.py`, `test_no_fullscreen.py`, `test_finish.py`: Sie starten cubi in einem Pseudo-Terminal und senden Tasten.

---

## Wenn nichts hilft

1. Den Befehl aus der Ansicht ohne cubi ausführen und die Rohausgabe lesen.
2. `cubi` im Checkout starten (`./bin/cubi`), um Installationsprobleme auszuschließen.
3. Version prüfen: [Releases](releases.md) und `git log` im Checkout.
4. Fehler mit Befehl, Meldung und Version als Issue in [pkr-lab/cubi](https://github.com/pkr-lab/cubi/issues) festhalten.
