## Agent skills

### Issue tracker

Issues are tracked in GitHub Issues for `lefeverd/backup-exporter`, via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Domain docs

Single-context layout — one `CONTEXT.md` + `docs/adr/` at the repo root (created lazily as needed). See `docs/agents/domain.md`.

## Development

Go is not installed on the host. Run Go commands inside the `go` toolbox:

```
toolbox run -c go make test
toolbox run -c go go mod tidy
```

Keep `go-version` in `.github/workflows/*.yml` in sync with the `go` directive in `go.mod`.
