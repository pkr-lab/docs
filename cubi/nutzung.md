# Nutzung

Code: [main.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/main.py)

---

## Aufruf

```bash
cubi                       # Hauptmenü
cubi <bereich>             # Bereich direkt öffnen
cubi --help                # Bereiche auflisten (auch: -h, help)
```

| Bereich | Aliase |
|---|---|
| `connect` | – |
| `argocd` | `argo` |
| `autosync` | `sync` |
| `feature-testing` | `branch`, `ft` |
| `scale` | – |
| `tailscale` | `ts` |
| `github` | `gh` |
| `kubernetes` | `k8s`, `kubectl` |
| `helm` | – |
| `tokens` | `token` |
| `login` | `auth` |

| Exit-Code | Bedeutung |
|---|---|
| 0 | normal beendet, auch nach Abbruch mit Esc |
| 1 | kein interaktives Terminal, oder ein Fehler beim Zeichnen des Hauptmenüs (zum Beispiel `Keine Einträge vorhanden`) |
| 2 | unbekannter Bereich |
| 130 | Abbruch mit Strg+C |

Ein Fehler **innerhalb** eines Bereichs (`CubiError`) beendet cubi nicht. `ui.guard()` zeigt ihn als roten Kasten „Fehler“ und kehrt zum Menü zurück.

---

## Bedienung in Kurzform

- ↑ und ↓ bewegen, Enter bestätigt, Esc geht eine Ebene zurück oder bricht ab, Strg+C bricht ab und beendet cubi.
- In Mehrfachauswahlen schaltet Space einen Eintrag um, `a` alle.
- In Tabellen lädt `r` neu. Die Fußzeile nennt die weiteren Tasten des Bereichs.
- `cubi autosync`, `cubi feature-testing` und `cubi scale` beenden cubi nach einer durchgeführten Änderung, damit die Shell sofort frei ist. Abbrechen führt zurück ins Menü.

Alle Tasten und das Inline-Verhalten: [Bedienung](bedienung.md).

---

## Typische Abläufe

### Warum ist eine App nicht grün?

```bash
cubi argocd
```

Probleme stehen oben. Enter auf der App → *Details* zeigt Meldung, Bedingungen und die betroffenen Ressourcen. *Diff* zeigt, was Git und Cluster unterscheidet, *Hard Refresh* verwirft den Manifest-Cache von ArgoCD. Dann `cubi kubernetes` für Pod-Status, Logs und Warnungen. Siehe [ArgoCD](argocd.md) und [Kubernetes](kubernetes.md).

### Änderung im Cluster testen, bevor sie in `main` landet

```bash
cubi feature-testing     # Apps auf den Feature-Branch stellen
# testen …
cubi feature-testing     # Auf main zurücksetzen
```

Details und Fallstricke: [ArgoCD-Features](argocd-features.md#feature-testing-branch-einer-app-umstellen).

### Eine App vorübergehend abschalten

```bash
cubi scale               # Auf 0 Pods stellen
cubi scale               # Pods wiederherstellen
```

Siehe [ArgoCD-Features](argocd-features.md#pods-skalieren-und-wiederherstellen).

### Token rotieren

```bash
cubi tokens
```

Das Token mit der kleinsten Restlaufzeit steht oben. Enter → *Rotieren*. Siehe [Tokens](tokens.md).

### Auf einen Host

```bash
cubi connect             # oder: cubi ts für Geräte im Tailnet
```

---

## Hinweise

- cubi verarbeitet selbst keine Passwörter oder Tokens. Logins laufen über `argocd`, `gh` und `tailscale`, die Token-Rotation über die Skripte von capulus-core. Das Register enthält nur Daten, keine Werte.
- Den ArgoCD-Status liest cubi direkt per `kubectl` aus dem Cluster. Er funktioniert deshalb auch, wenn der ArgoCD-Server nicht erreichbar ist.
- cubi committet nie etwas. Änderungen am Token-Register bleiben als Änderungen im Arbeitsverzeichnis von capulus-core, pro Rotation ein Pull Request.
- Wirken Änderungen nicht, hilft [Debugging und Fehlersuche](debugging.md).


---

## Fehlersuche

Fehler einzelner Bereiche stehen auf den Seiten der Bereiche ([ArgoCD](argocd.md#debugging-und-fehlersuche), [ArgoCD-Features](argocd-features.md#debugging-und-fehlersuche), [Connect](connect.md#debugging-und-fehlersuche), [Tailscale](tailscale.md#debugging-und-fehlersuche), [GitHub](github.md#debugging-und-fehlersuche), [Kubernetes](kubernetes.md#debugging-und-fehlersuche), [Helm](helm.md#debugging-und-fehlersuche), [Tokens](tokens.md#debugging-und-fehlersuche), [Login](login.md#debugging-und-fehlersuche)), allgemeine Meldungen und die Checkliste unter [Debugging und Fehlersuche](debugging.md).
