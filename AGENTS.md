# Repository Guidelines

## Project Structure & Module Organization
- Core runtime lives in `core/` and `main/` (CLI entry), with shared utilities in `common/`.
- Protocol implementations are under `proxy/`, transport layers under `transport/`, and feature glue in `features/` and `app/`.
- Configuration parsers and tooling sit in `infra/` (e.g., `infra/conf/` for config models, `infra/vprotogen/` for proto generation).
- Integration tests and reusable fixtures are in `testing/` (see `testing/scenarios/` for end-to-end coverage).

## Build, Test, and Development Commands
- `go build -o bin/xray ./main` builds the Xray binary; use Go 1.25+.
- `go test ./...` for a quick pre-push check; add `-count=1` when chasing flakes.
- Scenario suite: `go test ./testing/scenarios -count=1` exercises cross-module flows.
- Coverage (slow): `bash testing/coverage/coverall` generates merged `out/xray/cov/coverage.txt`.
- Proto regeneration (when editing `.proto`): `go run ./infra/vprotogen -pwd "$PWD"` with `protoc` in PATH.

## Coding Style & Naming Conventions
- Go defaults: tabs, gofmt on save (`gofmt -w` on touched files). Keep imports ordered by gofmt/goimports.
- Package names stay lower-case without underscores; exported types/functions need concise doc comments.
- Tests follow `*_test.go` with `TestName_Subtest` for tables. Prefer small, composable helpers in `testing/mocks/`.
- Avoid manual edits to `*.pb.go` or generated crypto assembly; regenerate via the documented tools instead.

## Testing Guidelines
- Write table-driven unit tests near the code they cover; favor deterministic inputs (no time.Sleep unless bounded).
- Prefer scenario additions in `testing/scenarios/` when behavior spans packages; isolate external dependencies with the mocks in `testing/mocks/`.
- Keep new tests tag-free unless matching existing tags (`json`, `coverage`); document any new tag in the PR.

## Commit & Pull Request Guidelines
- Commit messages are short and component-scoped (e.g., `proxy: fix udp leak`, `transport/tls: tighten pinning`); avoid long bodies unless clarifying risk.
- PRs should describe the problem, the fix, and impact. Include config snippets or reproduction steps when touching `infra/conf/` or protocol handling.
- List test evidence (`go test ./...`, targeted scenario commands) and note any skipped/slow cases.
- Attach screenshots only when changing user-facing CLI output; otherwise include before/after text if behavior shifts.

## Security & Configuration Tips
- Never check in secrets or example keys; scrub sample configs. Validate TLS/pinning changes with deterministic tests.
- Limit logging of user data; ensure new logs go through `common/log` levels and are guarded by debug/info checks.
