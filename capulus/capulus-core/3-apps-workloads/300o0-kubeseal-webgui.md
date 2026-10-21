# kubeseal-webgui — Secrets im Browser versiegeln

Weboberfläche, die einzelne Werte mit dem Zertifikat des TECH-Controllers zu einem `SealedSecret`-Wert verschlüsselt. Praktisch, wenn kein `kubeseal` oder kein Cluster-Zugriff am Arbeitsplatz verfügbar ist. Das Ergebnis ist derselbe Base64-Block wie bei `kubeseal --raw`.

> **Cluster:** TECH · Ordner `argocd/apps/tech/kubeseal-webgui/` · URL `http://kubeseal-webgui.tech.homeserver` (nur intern, kein TLS-Block im Ingress). `kubectl`-Befehle in diesem Doc gelten für den TECH-Cluster ([Zugriff je Cluster](../../capulus-ansible/a-betriebssystem/a0010-overview.md#kubectl-zugriff-je-cluster)).

---

## Architektur

| Teil | Wert |
|---|---|
| Chart | Wrapper um das Chart `kubeseal-webgui` 6.0.14 (`oci://ghcr.io/jaydee94/kubeseal-webgui/charts`, App 4.10.1) |
| Zertifikat | `sealedSecrets.autoFetchCert: true`: die Oberfläche holt das Zertifikat selbst vom Controller (`sealed-secrets-controller` im Namespace `sealed-secrets`) |
| Ingress | Klasse `traefik`, Host `kubeseal-webgui.tech.homeserver`, `tls.enabled: false`, API-Adresse `http://kubeseal-webgui.tech.homeserver` |
| Skalierung | eigenes Template `templates/hpa.yaml`: 1 bis 2 Replikas bei 75 % CPU, Scale-Down nach 300 s ([HPA](../b-kubernetes-gitops/b0040-hpa-autoscaling.md)) |
| Ressourcen | 20m CPU Request, 256 MiB Request und Limit |

Die Oberfläche kennt nur das **TECH**-Zertifikat. Werte für PROD oder ENTW lassen sich damit nicht versiegeln (dafür `kubeseal --cert <prod-zertifikat>` oder `scripts/reseal-for-prod.sh`, siehe [Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md)).

---

## Nutzung

1. `http://kubeseal-webgui.tech.homeserver` öffnen (Heimnetz oder Tailnet).
2. Scope wählen (Standard: strikt, Namespace und Secret-Name), Namespace und Secret-Name eintragen, den Wert eingeben.
3. Das Ergebnis (`AgB…`) in die `values.yaml` des jeweiligen Charts übernehmen (`encryptedData` o. Ä.), siehe die Anleitung der jeweiligen App, zum Beispiel [carplay-api: Ersteinrichtung](300d0-carplay-api.md#ersteinrichtung).

Der Klartext wird an die Oberfläche übertragen. Weil der Zugriff unverschlüsselt (HTTP) im internen Netz läuft, sollte man ihn nur von vertrauenswürdigen Geräten nutzen und besonders sensible Werte per CLI versiegeln.

---

## Fehlersuche

```bash
kubectl -n kubeseal-webgui get pods,ingress
kubectl -n kubeseal-webgui logs deploy/kubeseal-webgui --tail=100
```

| Symptom | Ursache | Lösung |
|---|---|---|
| Seite lädt nicht | Pod nicht bereit, Ingress oder DNS fehlt, kein VPN | `kubectl -n kubeseal-webgui get pods`, `nslookup kubeseal-webgui.tech.homeserver`, [DNS](../c-netzwerk-dns/c0000-dns-architecture.md) |
| „Zertifikat konnte nicht geladen werden“ oder leeres Ergebnis | Oberfläche erreicht den Controller nicht (falscher Name oder Namespace, NetworkPolicy) | `sealedSecrets.controllerName` und `controllerNamespace` in `values.yaml` prüfen, [Network Policies](../d-sicherheit/d0030-network-policies.md) |
| Ergebnis lässt sich im Cluster nicht entschlüsseln | falscher Name oder Namespace, oder der Wert gehört in einen anderen Cluster | Scope und Namen prüfen ([Sealed Secrets](../b-kubernetes-gitops/b0070-sealed-secrets.md#fehlersuche)) |
| Browser meldet gemischte Inhalte | API-Adresse in `api.url` passt nicht zum aufgerufenen Host (http/https) | `api.url` und Aufruf-URL angleichen |
| Pod startet nicht (`ImagePullBackOff`) | OCI-Registry nicht erreichbar, `pullPolicy: Always` | Netz prüfen, [Images](../f-cicd-automatisierung/f0060-build-images.md) |
| ArgoCD-Sync scheitert mit Chart-Fehler | `Chart.lock` oder OCI-Abhängigkeit nicht ladbar | [ArgoCD](../b-kubernetes-gitops/b0010-argocd.md) |

Allgemeine Fehlersuche: [zentrale Fehlersuche](../2-betrieb-hardware/20070-troubleshooting-zentral.md).
