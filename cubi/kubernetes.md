# Kubernetes

Pods, Nodes und Warnungen ansehen, Namespace und Kontext wechseln.

Code: [modules/kube.py](https://github.com/pkr-lab/cubi/blob/main/src/cubi/modules/kube.py) (alle Aufrufe gehen über `kubectl`)

---

## Überblick

`cubi kubernetes` (`k8s`, `kubectl`) arbeitet mit dem aktuellen `kubectl`-Kontext und -Namespace. Den Cluster der Homelab-Umgebung beschreibt [k3s](../capulus/capulus-core/b-kubernetes-gitops/b0000-k3s.md).

| Eintrag | Befehl | Inhalt |
|---|---|---|
| Pods | `kubectl get pods [-n <ns>\|-A] -o json` | Tabelle, alle 10 Sekunden neu |
| Nodes | `kubectl get nodes -o json` | Tabelle, alle 15 Sekunden neu |
| Warnungen | `kubectl get events -A --field-selector type=Warning --sort-by=.lastTimestamp` | einmalige Ausgabe |
| Namespace wechseln | `kubectl config set-context --current --namespace=<ns>` | schreibt in die kubeconfig |
| Kontext wechseln | `kubectl config use-context <name>` | schreibt in die kubeconfig |

Namespace und Kontext stehen als Hinweis im Menü (`aktuell: …`). Der Wechsel gilt auch außerhalb von cubi, weil er die kubeconfig ändert.

---

## Pods

Spalten: (Namespace bei „alle Namespaces“), Pod, Ready (`2/3`), Status, Restarts, Alter, Node. Mit `a` schaltet man zwischen aktuellem Namespace und allen Namespaces um.

Der Status wird wie bei `kubectl get pods` aus mehreren Feldern abgeleitet (`pod_status`): `Terminating` bei gesetztem `deletionTimestamp`, `Init:<Grund>` bei wartenden oder fehlgeschlagenen Init-Containern, sonst der Grund des wartenden oder beendeten Containers (`CrashLoopBackOff`, `ImagePullBackOff`, `OOMKilled`, …), sonst die Phase.

| Farbe | Bedeutung |
|---|---|
| grün | `Running` und alle Container bereit |
| gelb | `Running`, aber nicht alle bereit; `Pending`, `ContainerCreating`, `Terminating`, `PodInitializing`, `Init:…` |
| grau | `Completed`, `Succeeded` |
| rot | alles andere (`Error`, `CrashLoopBackOff`, `ImagePullBackOff`, `Failed`, …) |

Die Zusammenfassung zählt „N Pods · M mit Auffälligkeiten“ (rot oder gelb). Ein Pod-Menü bietet:

| Aktion | Befehl |
|---|---|
| Logs | `kubectl logs -n <ns> <pod> --all-containers=true --tail=300` |
| Logs folgen | `kubectl logs -f … --tail=100`, Abbruch mit Strg+C |
| Describe | `kubectl describe pod -n <ns> <pod>` |
| Shell | `kubectl exec -it … -- sh -c 'command -v bash … && exec bash \|\| exec sh'` |
| Löschen | `kubectl delete pod` nach Rückfrage |

*Löschen* ist die einzige zerstörende Aktion. Bei Pods, die ein Deployment oder StatefulSet verwaltet, startet Kubernetes den Pod neu.

## Nodes

Spalten: Node, Status (`Ready` grün, `NotReady` rot), Rollen (aus `node-role.kubernetes.io/*`), Version, interne IP, Alter. Enter zeigt `kubectl describe node`. Zum Cluster-Aufbau siehe [k3s](../capulus/capulus-core/b-kubernetes-gitops/b0000-k3s.md) und [Cluster Power Manager](../capulus/capulus-ansible/2-betrieb-hardware/20020-cluster-power-manager.md) (Worker werden nachts heruntergefahren, `NotReady` ist dann erwartbar).

---

## Debugging und Fehlersuche

Ladefehler erscheinen in der Tabelle als rote Zeile `✖ <Meldung>`. Mit `r` lädt man neu, die Tasten funktionieren weiter.

| Meldung | Ursache | Lösung |
|---|---|---|
| `kubectl ist nicht installiert oder nicht im PATH` | `kubectl` fehlt | installieren |
| `The connection to the server … was refused` / `Unable to connect to the server` | falscher Kontext, Cluster aus, VPN fehlt | `kubectl config current-context`, im Bereich *Kontext wechseln* korrigieren, [Tailscale](tailscale.md) prüfen |
| `error: You must be logged in to the server (Unauthorized)` | Zertifikat oder Token der kubeconfig abgelaufen | kubeconfig erneuern |
| `Error from server (Forbidden): pods is forbidden …` | keine Rechte im Namespace | anderen Namespace oder Kontext wählen, RBAC prüfen |
| `Zeitüberschreitung bei: kubectl get pods …` | API-Server langsam oder nicht erreichbar (Timeout 30 Sekunden) | Verbindung testen, `kubectl get --raw /readyz` |
| `Ungültige Ausgabe von: kubectl …` | Ausgabe war kein JSON (zum Beispiel Warnung vor dem JSON) | Befehl mit `-o json` direkt ausführen |
| `Keine Kontexte gefunden` | kubeconfig leer | `KUBECONFIG` prüfen |
| `Kontext konnte nicht gesetzt werden`, `Namespace konnte nicht gesetzt werden` | kubeconfig nicht schreibbar oder Name ungültig | Rechte der kubeconfig prüfen |
| `Keine Einträge vorhanden` | zum Beispiel Namespaceliste leer | Kontext prüfen |
| Tabelle zeigt `Keine Einträge` | im Namespace laufen keine Pods | `a` für alle Namespaces |
| Pod-*Shell* endet sofort | Image ohne `sh` (distroless) | `kubectl debug` außerhalb von cubi |
| *Logs* meldet `Keine Logs` | Container noch nicht gestartet | *Describe* → Events lesen |

Typische Pod-Fehlerbilder aus der Statusspalte:

| Status | Ursache | Weiter |
|---|---|---|
| `CrashLoopBackOff` | Container startet und beendet sich | *Logs* (bei Neustarts: `kubectl logs --previous`), *Describe* → Exit-Code |
| `ImagePullBackOff` / `ErrImagePull` | Image oder Tag fehlt, Registry nicht erreichbar | *Describe* → Events, Image-Name und Pull-Secret prüfen |
| `Pending` | keine passende Node, PVC nicht gebunden | *Describe* → Events (`FailedScheduling`), Node-Status im Bereich Nodes prüfen |
| `OOMKilled` | Speicherlimit überschritten | Limits im Chart, siehe [HPA und Ressourcen](../capulus/capulus-core/b-kubernetes-gitops/b0040-hpa-autoscaling.md) |
| `Init:Error` | Init-Container fehlgeschlagen | `kubectl logs <pod> -c <init-container>` |

Checkliste „Pod läuft nicht“:

1. Bereich Nodes: alle `Ready`?
2. Pod-Menü → *Describe*, unten bei Events.
3. *Logs*, bei Neustart zusätzlich `--previous`.
4. *Warnungen*: gibt es clusterweit Events?
5. Zum Dienst gehörende Doku, zum Beispiel [Dienste-Übersicht in capulus-core](../capulus/capulus-core/index.md).

Allgemeines: [Debugging und Fehlersuche](debugging.md).
