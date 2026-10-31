# MDM — Apple-Setup (manuelle Checkliste)

Apple erlaubt diese Schritte nur mit einem Browser-Login. Alles drumherum ist vorbereitet, jeder Schritt ist nur Hochladen und Herunterladen. Voraussetzung: der Stack läuft ([300s0](300s0-mdm-installation.md)), die n8n-Workflows sind importiert.

> **Hinweis zu den Screenshots:** Die Platzhalter `Screenshot: …` werden beim ersten Durchlauf durch echte Bilder ersetzt. Die Menünamen bei Apple ändern sich gelegentlich (Apple Business hieß früher Apple Business Manager). Maßgeblich ist der Sinn des Schritts.

Aufwand im Überblick: einmalig etwa eine Stunde, danach jährlich zwei kurze Verlängerungen ([300v0](300v0-mdm-betrieb.md#jährliche-verlängerung)).

| Schritt | Was | Wie oft |
|---|---|---|
| 1 | Apple Business: Organisation und Admin-Account | einmalig |
| 2 | Push-Zertifikat (mdmcert.download, identity.apple.com, n8n-Formular) | einmalig, dann jährlich |
| 3 | Enrollment-Profil versiegeln | einmalig |
| 4 | MDM-Server in Apple Business anlegen, DEP-Token einspielen | einmalig, Token dann jährlich |
| 5 | Server als Standard-MDM für iPads setzen | einmalig |
| 6 | Bestehende iPads mit Apple Configurator zu Apple Business hinzufügen | einmalig je iPad |
| 7 | Restliche n8n-Workflows aktivieren | einmalig |

---

## 1. Apple Business: Organisation und Admin-Account

Falls noch nicht vorhanden: Organisation bei Apple Business anmelden (D-U-N-S-Nummer und Verifizierung der Organisation, das dauert bei Apple einige Tage) und einen eigenen Admin-Account für das MDM anlegen. Dieser Apple Account wird später für das Push-Zertifikat **immer wieder** gebraucht. Zugangsdaten gehören in den Passwortmanager.

> Screenshot: Apple Business, Anmeldung und Verwaltung der Accounts

---

## 2. Push-Zertifikat

Der MDM-Server braucht ein von Apple ausgestelltes Push-Zertifikat (APNs). Apple verlangt dafür eine von einem MDM-Anbieter signierte Anfrage. Dafür dient der kostenlose Dienst mdmcert.download (nur für Organisationen, einmalige Registrierung mit E-Mail-Bestätigung unter `https://mdmcert.download/registration`).

1. **Anfrage erzeugen und senden.** Der API-Schlüssel des Dienstes steht in der Quelle `cmd/mdmctl/mdmcert.download.go` im Repo `micromdm/micromdm` (Konstante `mdmcertAPIKey`).
   ```bash
   MDMCERT_API_KEY=<Schlüssel> argocd/apps/tech/mdm/scripts/mdm-apns.sh request <registrierte-E-Mail>
   ```
   Das Skript erzeugt in `~/mdm-apple/apns/` den Push-Schlüssel `push.key` (geheim halten, wird später gebraucht), die Anfrage `push.csr` und ein Schlüsselpaar `pki.key`/`pki.crt` für die Antwort.
2. **Antwort entschlüsseln.** Die signierte Anfrage kommt per E-Mail als Anhang (`mdm_signed_request….plist.b64.p7`). Anhang speichern und entschlüsseln:
   ```bash
   argocd/apps/tech/mdm/scripts/mdm-apns.sh decrypt ~/Downloads/mdm_signed_request.….p7
   ```
   Ergebnis: `~/mdm-apple/apns/push.req`.
3. **Bei Apple hochladen.** `https://identity.apple.com` mit dem MDM-Admin-Account öffnen, **Create a Certificate** wählen, `push.req` hochladen, das Zertifikat als `push.pem` herunterladen.
   > Screenshot: Apple Push Certificates Portal mit Upload der signierten Anfrage
4. **In n8n einspielen.** Das Formular `https://n8n.prod.homeserver/form/mdm-verlaengerung` öffnen (Benutzer `mdm-admin`, Passwort `form-password` aus `~/mdm-secrets/credentials.env`), **Push-Zertifikat erneuern** wählen, `push.pem` und `push.key` hochladen. n8n prüft, ob Zertifikat und Schlüssel zusammengehören, spielt beides in NanoMDM ein und zeigt das gespeicherte Ablaufdatum.
   > Screenshot: n8n-Formular „MDM Verlängerung einspielen“ mit Erfolgsmeldung

---

## 3. Enrollment-Profil versiegeln

Das Profil enthält das Push-Topic aus dem Zertifikat, die SCEP-URL und die MDM-URL.

```bash
argocd/apps/tech/mdm/scripts/mdm-enrollment-profile.sh ~/mdm-apple/apns/push.pem
```

Das Skript liest das Topic (`com.apple.mgmt.External.…`) aus dem Zertifikat, schreibt `~/mdm-apple/enroll.mobileconfig` (zum Testen) und versiegelt das Profil in `values.yaml` (`secrets.enrollmentProfile`). Danach PR erstellen und mergen. Der Pod `mdm-enroll` entsteht, der Ingress bekommt den Pfad `/mdm/enroll`.

Prüfen von außen:
```bash
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' https://mdm-prod.pke-lab.de/mdm/enroll
```
Erwartet: `200 application/x-apple-aspen-config`.

---

## 4. MDM-Server in Apple Business anlegen, DEP-Token einspielen

1. **Öffentlichen Schlüssel erzeugen** (der private Schlüssel bleibt in der NanoDEP-Datenbank und im Backup):
   ```bash
   argocd/apps/tech/mdm/scripts/mdm-dep-keypair.sh
   ```
   Ergebnis: `~/mdm-apple/dep-public-cert.pem`.
2. In Apple Business: **Einstellungen → Geräteverwaltung (MDM-Server) → MDM-Server hinzufügen**, Namen `Capulus MDM` vergeben, `dep-public-cert.pem` hochladen, speichern.
   > Screenshot: Apple Business, MDM-Server hinzufügen mit Zertifikat-Upload
3. Beim neuen MDM-Server **Token herunterladen** (Datei `….p7m`).
   > Screenshot: Apple Business, Token herunterladen
4. Im n8n-Formular **DEP-Token erneuern** wählen und die `.p7m`-Datei hochladen. n8n entschlüsselt den Token in NanoDEP, prüft die Verbindung zu Apple Business (Servername, Organisation), zeigt das Ablaufdatum und legt beim ersten Mal **automatisch** das DEP-Enrollment-Profil an und setzt es als Assigner. Ab jetzt startet `mdm-depsyncer` von selbst.

---

## 5. Server als Standard-MDM für iPads setzen

In Apple Business: **Einstellungen → Geräteverwaltung → Standard-Geräteverwaltung (Standardserver für neue Geräte)**, für **iPad** den Server `Capulus MDM` wählen. Neu gekaufte oder neu zugeordnete iPads landen damit automatisch bei diesem MDM-Server.
> Screenshot: Apple Business, Standard-MDM-Server pro Produktart

---

## 6. Bestehende iPads zu Apple Business hinzufügen

Bereits genutzte iPads (nicht über den Händler in Apple Business eingetragen) müssen einmalig mit **Apple Configurator** (Mac, App Store, kostenlos) hinzugefügt werden. Das iPad wird dabei zurückgesetzt.

1. iPad per Kabel an den Mac anschließen, in Apple Configurator auswählen.
2. **Hinzufügen → Zu Apple Business hinzufügen** wählen (Apple Configurator führt durch Anmeldung mit dem Admin-Account und Zuordnung).
3. Das iPad wird gelöscht, neu gestartet und erscheint in Apple Business unter **Geräte**. In der Geräteliste den MDM-Server `Capulus MDM` zuordnen, falls der Standard nicht greift.
4. `depsyncer` übernimmt die Seriennummer beim nächsten Lauf (höchstens 10 Minuten), weist das DEP-Profil zu und n8n meldet, falls etwas offen bleibt.
   > Screenshot: Apple Configurator, „Zu Apple Business hinzufügen“

Hinweis: Ein iPad, das schon einmal anderswo in Apple Business eingetragen war, muss dort zuerst freigegeben werden.

---

## 7. Restliche n8n-Workflows aktivieren

Bis zu diesem Punkt waren Formular, Geräte-Workflow, Lock-Screen-Workflow und Backup-Check aktiv. Jetzt die übrigen aktivieren (sie würden vorher nur Alarme für noch nicht eingerichtete Teile senden):

```bash
N8N_API_KEY=<API-Schlüssel> argocd/apps/tech/mdm/scripts/mdm-n8n-setup.sh --activate
```

---

## End-to-End-Test mit einem zurückgesetzten Test-iPad

Diesen Test führst du selbst durch. Er prüft die gesamte Kette: Apple Business → DEP → Enrollment-Profil → SCEP → MDM → n8n → Sperrbildschirm.

1. Ein Test-iPad in Apple Business hinzufügen (Schritt 6) und warten, bis die Seriennummer in der Geräteliste mit dem Server `Capulus MDM` steht (bis zu 10 Minuten bis zur Zuweisung des DEP-Profils).
2. In den Logs prüfen, dass die Zuweisung stattfand:
   ```bash
   kubectl -n mdm logs deploy/mdm-depsyncer --tail=20
   ```
   Erwartet: Zeile `profile assigned` mit `success=1`.
3. iPad zurücksetzen (Einstellungen → Allgemein → iPad übertragen/zurücksetzen → Alle Inhalte & Einstellungen löschen) und mit WLAN verbinden. Während der Einrichtung erscheint der Hinweis **Fernverwaltung**: Die Anmeldung ist verpflichtend (`is_mandatory`), der MDM-Server nicht entfernbar.
4. Der Einrichtungsassistent überspringt die in `profiles/dep-enrollment-profile.json` genannten Schritte (Apple-ID, Diagnose, Siri und weitere).
5. Beobachten, was passiert:
   ```bash
   kubectl -n mdm logs deploy/mdm-nanomdm -f
   ```
   Erwartet: `Authenticate`, `TokenUpdate`, danach die Befehle aus n8n.
6. **Sperrbildschirm prüfen:** iPad sperren (Seitentaste). Unter dem Sperrbildschirm erscheint die Nachricht aus `automation.lockScreenText` (bzw. dem zuletzt im Formular gesetzten Text).
7. Datenbank-Sicht auf den Status des Profils (`queued` → `installed`):
   ```bash
   kubectl -n mdm exec deploy/mdm-postgres -- psql -U postgres -d nanomdm -c "select * from automation.device_profile"
   ```
8. Text ändern: Formular `https://n8n.prod.homeserver/form/mdm-lockscreen`. Nach wenigen Sekunden zeigt das iPad den neuen Text.
9. Wiederholter Check-in (iPad kurz aus- und einschalten) darf **kein** zweites Profil senden: `automation.device_profile` bleibt bei `version = 1` bzw. der aktuellen Version, in der ntfy-App kommt keine Meldung.

Vorher einmal prüfen, ob Cloudflare für die MDM-Pfade nichts blockt (WAF-Regel „Skip“ für `mdm-prod.pke-lab.de` auf `/mdm`, `/mdm/enroll`, `/scep`, Begründung und Hinweise: [600d0](../6-hintergruende/600d0-mdm.md#authentifizierung-der-geräte-hinter-cloudflare)). Während des Tests Cloudflare → Sicherheit → Ereignisse beobachten.

Wenn etwas hängt: [300v0, Troubleshooting](300v0-mdm-betrieb.md#troubleshooting). Offen bis zu diesem ersten echten Test ist, ob Apple das unsignierte Enrollment-Profil beim DEP-Abruf akzeptiert und ob die Authentifizierung über den `Mdm-Signature`-Header durch den Cloudflare-Tunnel hindurch funktioniert (Begründung: [600d0](../6-hintergruende/600d0-mdm.md)).
