# Releases

Releases entstehen automatisch aus den Commit-Nachrichten, wie in allen Repos von pkr-lab ([github-workflows](../../github-workflows/index.md)). Commitlint prüft auf Pull Requests das Format Conventional Commits, der Push auf `main` startet `release.yml` (semantic-release).

## Image

`build-image.yml` baut bei jeder Änderung an Code, `go.mod`, `go.sum` oder Dockerfile das Image und pusht `ghcr.io/pkr-lab/capulus-operator:sha-<commit>`. Der Tag wird von Hand in `argocd/apps/tech/capulus-operator/values.yaml` eingetragen. Eine automatische Übernahme über einen Pull Request gibt es noch nicht.

## CI

`ci.yml` prüft `gofmt`, `go vet`, `go test`, dass die generierten Dateien (`api/`, `config/`) eingecheckt sind, `go mod tidy`, gitleaks und den Docker-Build.
