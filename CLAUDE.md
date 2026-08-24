# jai

Go CLI that syncs Jira Cloud to local SQLite and exposes it via SQL, CLI, and TUI.

## Build & Test

```bash
make build                              # compile ./jai (requires CGO + fts5)
make test                               # go test -tags fts5 ./...
make vet                                # go vet
make check                              # vet + test
make lint                               # golangci-lint (v2, config in .golangci.yml)
```

**Critical**: ALL go commands need `-tags fts5` — the SQLite schema uses FTS5 virtual tables. Without it, tests fail with `no such module: fts5`. The Makefile handles this automatically.

To run a single package's tests:
```bash
go test -tags fts5 ./internal/db/...
go test -tags fts5 -run TestFunctionName ./internal/cli/...
```

## Code Style

- **Conventional commits**: `feat:`, `fix:`, `chore:`, `refactor:`, `test:`, `docs:`
- **Linter**: govet, staticcheck, unused, ineffassign (see `.golangci.yml`)
- **Error handling**: Return `fmt.Errorf("context: %w", err)` — wrap, don't swallow
- **Output modes**: All commands support `--json` for agent consumption. Human output uses lipgloss tables. Use `output.JSON()`/`output.Table()` from `internal/output/`
- **Write operations**: Default is write-through to Jira. `--queue` defers to `pending_changes` table, flushed by `jai push`

## Architecture

```
cmd/jai/main.go          → entrypoint
internal/
  cli/                   → cobra commands (one file per command)
  config/                → YAML config with ${ENV_VAR} substitution
  db/                    → SQLite: schema, migrations, upserts, field_map
  jira/                  → HTTP client, pagination, ADF→plaintext, writes
  sync/                  → incremental/full sync, denormalization, deletions
  query/                 → SQL execution, template vars, result formatting
  tui/                   → bubbletea full-screen app
  output/                → JSON/table formatting, schema introspection
```

**DB-first**: SQLite is the source of truth for reads. No read command hits Jira directly. All Jira fields stored as raw JSON and denormalized into queryable columns. Custom field names auto-discovered via `field_map` table.

**Reference docs** (read before implementing features):
- `docs/spec.md` — data models, API surface, TUI design, sync engine
- `docs/plan.md` — phased task breakdown (source of truth for what to build)
- `docs/idea.md` — high-level overview and motivation

## Development Workflow

- Proceed **autonomously** unless you hit a genuine blocker
- Implement independent tasks **in parallel** using sub-agents
- Create a commit after every meaningful change
- Track work in **Beads** (`bd`) — run `bd ready` for next tasks

## Code Navigation

Prefer **LSP** over grep/read once you have file coordinates:

| Task | Use | Not |
|------|-----|-----|
| Find definition | `goToDefinition` | `grep "func X"` |
| Find all usages | `findReferences` | `grep "X"` |
| Trace call chains | `incomingCalls`/`outgoingCalls` | manual grep |
| Check type/signature | `hover` | reading whole file |
| List file symbols | `documentSymbol` | skimming with Read |
| Workspace symbol search | `workspaceSymbol` | Glob + Grep |
| Find implementations | `goToImplementation` | grep method names |

Grep is still right for: initial search without coordinates, error messages, config values, comments, non-code files.

## Session Completion

Work is **not done** until `git push` succeeds.

```bash
make check                    # tests + vet pass
git add <files> && git commit -m "feat: ..."
git pull --rebase
bd dolt push
git push
```
