# Releases

Das Repo `pkr-lab/.github` nutzt seinen eigenen Release-Workflow (`self-release.yml` ruft `release.yml` lokal auf). Versionen und Tags (`v1.0.0`, `v1.0.1`, …) stehen im [CHANGELOG.md](https://github.com/pkr-lab/.github/blob/main/CHANGELOG.md) und bei den [GitHub Releases](https://github.com/pkr-lab/.github/releases).

Die Aufrufer in den anderen Repos verweisen auf den Commit-SHA eines Release-Tags. Renovate öffnet einen PR, wenn ein neues Release erscheint. Änderungen am Workflow:

| Änderung | Commit-Typ | Wirkung |
|---|---|---|
| neue Eingabe, neues Verhalten ohne Bruch | `feat` | Minor |
| Fehlerbehebung | `fix` | Patch |
| geänderte oder entfernte Eingabe | `feat!` | Major, alle Aufrufer anpassen |

Beschreibung des Ablaufs allgemein: [Releases in capulus-core](../capulus-core/releases.md).
