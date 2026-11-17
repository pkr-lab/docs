# isc-manager-bot

HTTP-API, die eine Person aus SEWOBE (`https://mv.dlrg.net`) in das DLRG ISC (`https://dlrg.net`, App „Personen“) überträgt. Steuert headless Chromium mit Playwright. n8n ruft den Dienst auf, im n8n-Pod läuft kein Browser.

Vollständige Doku (Ablauf, Abgleich, Mapping, API, Secrets, Deployment, n8n-Workflow, Troubleshooting, Risiken): [capulus-core/3-apps-workloads/300x0-isc-manager-bot.md](../3-apps-workloads/300x0-isc-manager-bot.md).

## Aufbau

| Pfad | Inhalt |
|---|---|
| `src/` | Dienst: HTTP-API (`server.ts`), Lauf-Orchestrierung (`runner.ts`), Anmeldung per TOTP (`totp.ts`), Validierung |
| `src/sync/` | Mapping SEWOBE → ISC, Abgleich (Id, Name + Geburtsdatum), Änderungsplan |
| `src/sewobe/`, `src/isc/` | Schritte je System, alle Selektoren in `selectors.ts` |
| `test/unit/` | Unit-Tests: Mapping, Datum, Abgleich, Plan, TOTP (RFC-6238-Vektoren), Validierung, Logger |
| `test/e2e/` | Playwright-Tests gegen Mock-SEWOBE und Mock-ISC, kein Zugriff auf die echten Systeme |
| `scripts/smoke.ts` | Dry-Run gegen die echten Systeme (`npm run smoke`) |
| `templates/` | Helm-Chart: Deployment, Service, ConfigMap, SealedSecret, NetworkPolicies |
