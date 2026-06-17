# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository overview

Teleport is an identity-aware access proxy and CA for SSH, Kubernetes, databases, RDP, web apps, cloud consoles, and MCP servers. It ships as a single Go binary (`teleport`) plus user-facing CLIs (`tsh`, `tctl`, `tbot`), an Electron desktop app (Teleport Connect), and a React web UI served by the proxy.

This is a polyglot monorepo: Go (primary), TypeScript/React (web UI and Teleport Connect), Rust (RDP client, IronRDP wasm, FIDO2 bindings), and BPF (Linux session enhanced recording).

Also see `AGENTS.md` — review guidelines (focus on critical security/reliability/performance issues; ignore style nits unless tied to a real failure).

## Top-level layout

- `lib/` — the bulk of Go server code, organized by subsystem (≈100 packages: `auth/`, `backend/`, `cache/`, `kube/`, `srv/`, `services/`, `web/`, `tbot/`, `vnet/`, ...). When changing behavior, the implementation usually lives here, not under `tool/`.
- `tool/` — CLI entrypoints: `teleport` (daemon), `tsh` (user client), `tctl` (admin client), `tbot` (machine ID), `teleport-update`, `fdpass-teleport`. These are thin wrappers around `lib/`.
- `api/` — public Go client SDK, a separate Go module under Apache 2.0 (the rest of the repo is AGPLv3). Consumers import `github.com/gravitational/teleport/api/...`.
- `proto/` and `api/proto/` — Protocol Buffer definitions. Generated code lives under `gen/` and `api/gen/`.
- `web/packages/` — pnpm workspace: `teleport` (web UI), `teleterm` (Electron app, "Teleport Connect"), `shared`, `design`, `build`.
- `e/` — git submodule for Teleport Enterprise code. **Empty in OSS clones** — anything `_imports.go` references but you cannot find is likely in `e/`. `e_imports.go` lists the enterprise import surface.
- `integrations/` — separate Go modules: `operator` (Kubernetes operator), `terraform` (Terraform provider), `event-handler`, `access/*` (access request plugins like Slack/Jira/PagerDuty), `kube-agent-updater`.
- `integration/` (singular) — in-tree Go integration tests; require a built binary and run `teleport` end-to-end.
- `e2e/` — Playwright-style browser end-to-end tests for the web UI.
- `bpf/` — eBPF programs for enhanced session recording (Linux only).
- `docs/` — product documentation. Per AGENTS.md, consult this when working on a product area to understand intended usage.
- `rfd/` — Request For Discussion design documents (numbered, e.g. `0001-testing-guidelines.md`). Browse for architectural intent.
- `build.assets/` — Dockerized build tooling, pinned tool versions in `versions.mk`.

## Build & run

The `Makefile` is the canonical entrypoint; many targets shell out to `build.assets/` for hermetic dockerized builds.

- `make full` — build all Go binaries with web UI baked in. Output in `build/`.
- `make build/tsh`, `make build/teleport`, etc. — build a single binary.
- `make -C build.assets build-binaries` — dockerized build, no local toolchain required.
- `make teleport-hot-reload` — runs the daemon under CompileDaemon, rebuilding on Go file changes. Pass `TELEPORT_ARGS='start --config=...'` to customize.
- `make grpc` — regenerate protobuf code (dockerized). Run after editing any `.proto`.
- `make protos/all` — build + lint + format protos directly (requires local `buf`).

Build flags worth knowing (toggle via env or make args): `FIPS=yes`, `FIDO2=dynamic|static|off`, `PIV=yes`, `BPF=yes`. See top of `Makefile` for the full message banner.

## Tests

- `make test` — runs everything: `test-helm test-sh test-api test-go test-rust test-operator test-terraform-provider`. Heavy.
- `make test-go` — Go unit tests (composed of `test-go-unit test-go-touch-id test-go-vnet-daemon test-go-tsh test-go-chaos`).
- `make test-go-unit` — main Go unit suite. Override `SUBJECT=./lib/foo/...` to scope, or `FLAGS='-run TestName -v'` for a single test. Default flags are `-race -shuffle on`.
- `make test-api` — tests for the `api/` module. Override `SUBJECT` similarly.
- `make integration` — in-tree integration tests under `./integration/...`. Requires built binaries (it depends on `session/reexec/embed/sessionhelper`). Slow.
- `make test-rust` — `cargo test`.
- `make test-operator`, `make test-terraform-provider`, `make test-kube-agent-updater` — per-integration module tests.

Single Go test: `go test -run TestName -v ./lib/path/...` works directly when you don't need the Makefile's webassets/rdpclient preconditions.

Web/JS:
- `pnpm test` (from repo root) — Jest unit tests for the web UI.
- `pnpm tdd` — Jest in watch mode.
- `pnpm test-update-snapshot` — refresh snapshots (use after adding audit events, see `web/README.md`).
- `pnpm storybook` — interactive component browser; needs `web/certs/` (see `web/README.md` for mkcert setup).

## Lint & format

- `make lint` — composite: `lint-api lint-go lint-kube-agent-updater lint-tools lint-protos lint-no-actions`.
- `make lint-go` — golangci-lint (config: `.golangci.yml`). Pass `GO_LINT_FLAGS=...` to override.
- `make fix-imports` — apply Go import ordering.
- `pnpm lint` — `oxlint` + format check for JS/TS.
- `pnpm format` / `pnpm format-check` — `oxfmt` over `web/`, `e/`, `e2e/`.
- `pnpm type-check` — TypeScript (using `tsgo`, the native preview compiler). `pnpm type-check-legacy` falls back to `tsc`.
- `make lint-protos` (or `make protos/lint`) — buf lint.

## Web UI development

The web UI is a pnpm workspace; activate with `corepack enable pnpm && pnpm install` from the root. Node version is pinned in `build.assets/versions.mk` (`make -C build.assets print-node-version`).

- `pnpm start-teleport` — Vite dev server. Set `PROXY_TARGET=host:port` to proxy API calls to an existing cluster; without it, defaults to `https://localhost:3080`. Requires `web/certs/` (see `web/README.md`).
- Built UI is embedded into the `teleport` binary at build time. Set `DEBUG=1` when starting `./build/teleport` to load assets from disk instead, so UI changes don't require a rebuild.
- When adding a new audit event, update both the Go side (`events.proto`) **and** the web UI in five places — see the `web/README.md` "Adding an Audit Event" checklist; run `make audit-event-reference` after.

## Working with the Enterprise submodule

`e/` is a git submodule pointing at the private enterprise repo and is empty for OSS contributors. Code in `lib/` and `tool/` is built to compile without it — guarded by build tags or stubbed via `e_imports.go` / `webassets_*` files. If a symbol seems to vanish or a build target references `*-ent` / `*-e`, it lives in `e/`.

## Protocol Buffers

All API/service definitions are in `proto/` and `api/proto/`. After any `.proto` edit run `make grpc` to regenerate Go stubs (and TS stubs for the web UI). The hot-reload loop will pick up regenerated `.go` files automatically.

## Code-review focus (from AGENTS.md)

When reviewing, ignore style and micro-perf. Surface: authz/authn bypasses, secret leakage, injection (SQL/command/template/path/SSRF), crypto misuse, privilege escalation, data corruption, concurrency races, and reliability regressions (panics, deadlocks, unbounded retries, nil derefs).
