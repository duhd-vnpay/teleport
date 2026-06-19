# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Language

Always communicate with the user in Vietnamese (tiếng Việt). Code, comments, commit messages, PR titles/descriptions, and changelog entries remain in English to match upstream Teleport conventions.

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

---

## Defensive engineering rules (mandatory)

These rules counteract known model-degradation patterns. Violating any is a hard failure, not a style preference.

### 1. Read-First Protocol
- Read the target file + grep references BEFORE any Edit. Never edit a file you haven't read in this conversation.
- For renames/refactors: grep usages across the codebase first (incl. `e_imports.go` for enterprise-only references).
- Target Read:Edit ratio ≥ 5:1.

### 2. No "simplest fix" shortcuts
When you catch yourself thinking "the simplest approach is...", STOP. Evaluate at least 2 approaches and pick the CORRECT one, not the fastest.

### 3. Surgical edits over rewrites
- Default to Edit (search/replace), not Write, for existing files. Write only for new files or >70% rewrites — and explain why.
- **Don't "improve" adjacent code** — no refactor of nearby code unless asked. AGPL header, import order, formatting of unrelated lines: leave alone.
- **Match existing style** — quote style, naming, slog vs. fmt, error wrapping (`trace.Wrap`) of the surrounding file.
- **Remove only your orphans** — clean up imports/vars you yourself orphaned; don't delete pre-existing dead code unless asked.
- **Don't fix unrelated bugs** — mention them, don't fix them in scope.
- Test: every changed line must trace directly to the user's request.

### 4. Plan first, surface assumptions
- Non-trivial task → write a short plan or spec before coding (goal + verify criteria + steps, 1 paragraph). Trivial → 1-2 line plan is fine.
- State assumptions explicitly before implementing (format, scope, data shape).
- Ambiguous request → present 2-3 interpretations and ask, don't silently choose.
- If a simpler approach exists, propose it and push back when warranted.

### 5. Simplicity first — do not overbuild
- Minimum code that solves the asked problem. Nothing speculative.
- No features beyond ask. No abstractions for single-use code. No "flexibility"/"configurability" not requested. No error handling for impossible scenarios.
- **Half-length test**: if you can rewrite it in half the lines and still be correct, rewrite.
- **<100 LOC heuristic** for typical tasks, **<1000 LOC** for complex features. Exceeding either is a signal of scope creep, premature abstraction, or solving the wrong problem.
- Typical anti-pattern: a discount-calculation does NOT need a Strategy pattern + abstract base + config object — `calculate(amount, percent)` suffices. Refactor when complexity actually appears, not before.

### 6. Follow project instructions
- Re-read `AGENTS.md` (review guide), nearest `CLAUDE.md`, and any relevant `rfd/` design doc before non-trivial work.
- Don't skip validation steps. Don't ignore conventions even when they look stylistic.

### 7. Verify before claiming done — hawk-eye throughout
- Hawk-eye DURING work, not only at the end. After each Edit: re-read the diff, check edge cases/assumptions/trade-offs, run lint/test mid-stream when changing multiple files. Never accept LLM/tool output blindly.
- Run verification BEFORE saying "done":
  - Go: `make lint-go GO_LINT_FLAGS='--new-from-rev=HEAD~'` and `go test -run TestName ./lib/path/...`
  - Go (full unit suite for a package): `make test-go-unit SUBJECT=./lib/path/...`
  - Protos: `make grpc` after `.proto` edits
  - Web: `pnpm lint && pnpm type-check && pnpm test`
- Transform vague tasks into verifiable goals: "fix bug X" → "write test reproducing X → make it pass → verify no regressions". "Add feature Y" → "tests for Y → implement → all pass".
- **Goal-driven looping**: success criteria first (test passes / lint clean / specific output) → implement → run verify → fail → fix → retry. Never declare done from "my logic looks right"; declare done from verify-command evidence.

### 8. Gather info, don't guess
- When uncertain, READ MORE CODE instead of reasoning in circles.
- If you write "oh wait" / "actually" / "let me reconsider" more than twice in a turn — stop talking, start reading.
- Reasoning loops = insufficient context, not insufficient thinking.

### 9. Codebase is the single source of truth
- The codebase (Go source, `.proto`, `examples/chart/*/values.yaml`, terraform, helm) is authoritative for intent. Runtime/live state is a projection of it, not the reverse.
- When you observe drift between live and code (a `kubectl patch` that's not in chart, a tctl resource that's not in YAML), surface and resolve it — don't silently work around.
- Memory does not replace code. Before recommending from memory, verify the file/flag/symbol still exists via grep.
- Canonical docs to keep in sync when changing related code:
  - `README.md` (developer overview, build commands)
  - `CHANGELOG.md` (per release)
  - `docs/` (per product area — Teleport's user-facing docs)
  - `rfd/` (new design doc when introducing/changing architecture)
  - For a new audit event: the 5-place web-UI checklist in `web/README.md` PLUS `make audit-event-reference`

### 10. Sub-agent decomposition — keep main context clean
- Delegate research / exploration / multi-file survey to a sub-agent (`Agent` tool) to keep the main context tight.
- Use sub-agents for: open-ended search >3 rounds, cross-file refactor impact analysis, parallel independent reads.
- Don't use sub-agents when the target is already known (path/symbol) — direct `Read`/`Grep` is faster.
- **Parallel when independent**: multiple `Agent` calls in **one message** to run concurrently. Sequential only when there's a data dependency.
- **Brief like a new colleague**: state goal + known context + ruled-out paths + desired response format. Terse command-style prompts produce shallow generic work.
- **Don't delegate understanding**: sub-agents return findings; the main agent synthesizes and decides. Never write "based on your findings, implement the fix" — that's delegating the judgment call.
- **Trust but verify**: a sub-agent's summary describes intent, not execution. When it writes code, inspect the actual diff before reporting done.

### 11. Code-review workflow — evidence + multi-angle fan-out
Every PR must be **verifiable** and **auditable** before merge. Agent self-review is a blind spot; use an independent reviewer (different sub-agent or person) for agent-authored code.

Scale review depth to size/criticality (also see Rule #5):

| Scope | Checklist |
|---|---|
| <50 LOC + trivial (typo/doc/comment) | verify command + one-line "why" |
| 50-300 LOC + normal feature/fix | pre-PR checklist (11.1) + careful test diff (11.2) |
| >300 LOC OR security/auth/migration/RBAC/billing/cron/CI config/`e/` submodule | full + multi-angle fan-out (11.3) + ≥1 distinct AI reviewer (11.5) |

#### 11.1 Pre-PR checklist
- **Evidence**: paste real `command + output`. Not "ran tests, all pass" — that's not evidence.
- **Decision log** (3-5 lines): assumptions surfaced, alternatives ruled out, concrete reason. Trivial PR can skip.
- **Scope = PR description**: the diff touches only what the title/body promises. Orphan file changes → split or reject.
- **Rollback plan**: 1-2 lines — revert command + the post-merge signal (metric, log, user report) that says it failed.
- **Reproducibility**: for UI / integration / infra changes → run locally and paste output. For Teleport: `make integration` slice, `tsh` exercise, web UI screenshot.

#### 11.2 Test review — read the test diff harder than the code diff
Agents often "cheat to make tests pass" — this is the highest-value failure mode to catch. For each changed test, ask: "Does this test still FAIL if I plant a real bug in the implementation?" If unsure, plant the bug, run the test, confirm it fails.

Reject immediately on any of:
- **Loosened assertion**: `require.Equal(t, 5, x)` → `require.NotNil(t, x)`, `>0` → `>=0`
- **Mock the whole module under test** — test becomes hollow
- **Expected value modified to match buggy output** (test follows the bug, not the spec)
- **New `t.Skip`/`SkipNow`/`@Ignore`/`it.skip`/`xit`** without an explicit ticket reason
- **Swallowed errors** in test body (`if err != nil { return }` instead of `require.NoError(t, err)`)
- **Happy path only** when the feature has error/edge branches
- **Snapshot/golden file accepted without inspecting the diff content**
- **Coverage threshold lowered** in CI config

#### 11.3 Multi-angle fan-out (for ≥300 LOC or high-criticality)
Spawn parallel sub-agents (Rule #10 — multiple `Agent` calls in one message), each with a **distinct** lens, no redundancy:
- **Correctness** — logic bug, off-by-one, race, null/empty/oversize input, error path coverage
- **Security** — injection (SQL/cmd/path/SSRF), auth bypass, secret leak (env/log), RBAC scope, crypto misuse, prompt-injection if LLM-adjacent
- **Test quality** — anti-patterns from §11.2, coverage gap for new code paths
- **Architecture** — consistency with codebase pattern, abstraction proportional to scope (no overkill)
- **Performance** — N+1 query, blocking call in async/hot path, unbounded allocation, missing index

Main agent then **synthesizes**: read all findings, dedupe overlaps, classify severity (block/warn/nit), reply on the PR with a summary plus raw per-angle findings attached. Never write "based on your findings, implement the fix" — Rule #10 forbids delegating understanding.

#### 11.4 CI gating — necessary, not sufficient
"CI green" validates what tests cover, not completeness. Watch for: skip-flaky-instead-of-fix, coverage threshold dropped, narrowed CI scope (unit only, no integration), mocking external deps so tests pass but prod won't.

When diff touches `.github/workflows/*` or any CI config: read it carefully. Did the test suite shrink? Did a stage get skipped? Did a threshold drop?

#### 11.5 Multi-reviewer protocol for high-stakes
High-stakes = prod incident fix, security patch, data migration, cron schedule change, billing, multi-tenant isolation, RBAC, infrastructure (helm/terraform), anything under `lib/auth/*`, `lib/services/*`, or modifying `e/` integration points.

- ≥1 distinct AI reviewer (different sub-agent prompt, or human). Diversity > redundancy.
- Adversarially prompt: "**Default to refuted=true if uncertain**" — counters confirmation bias.
- N=3 reviewers → a finding survives only with ≥2 agreement.
- Human integrates and decides — no auto-merge from majority vote.

### 12. Verify identities from source — never infer one identifier from an adjacent value

A system's **identity** (cluster_name, CA pin/fingerprint, host UUID, account/project/tenant ID, namespace, region, DB name) is NOT derivable from a **related-but-distinct** value (public address, DNS name, IP, proxy URL, repo name). They are different fields with different meanings — inferring one from the other is a hard failure, not a shortcut.

- **Capture before you bake.** Before writing any identifier into config / plan / code / helm values, read its REAL value from the authoritative live source (`tctl status`, `kubectl get`, cloud console, the API) and cite the command + output in the plan/PR. Never type an identifier you have not verified this session.
- **Address ≠ identity.** A public address/DNS name is what clients *connect to*; the cluster identity (`cluster_name`) is what *signs the CA*. Copying the address into an identity field silently generates a different trust root.
- **Label the distinction.** In config + comments, mark which fields are addresses and which are identities, so the difference can't be lost in a later rename/refactor.
- **"Preserve X" migrations need a loud gate.** When a migration claims to preserve an identity (CA, cluster_name), pin the source value and add a gate that FAILS LOUDLY on mismatch (e.g. CA-pin equality). That gate must NOT be skippable (`STAGING_MODE`, `--force`, `|| true`) on the one path where it is the last line of defense — a gate bypassed in the only run that would catch the error catches nothing.
- **Trigger:** if you think "X is probably the same as Y", or you are about to copy an address into an identity field — STOP, read the source, verify.

> Cautionary incident (2026-06-19): helm `clusterName` was set to the public address `teleport.x.vnshop.cloud` instead of the real `cluster_name: teleport.9ping.cloud` (plainly visible in `tctl status`). The K8s cluster therefore generated a fresh CA (pin `465e382b…` ≠ production `0bab7502…`); the staging dry-run had skipped GATE 1 via `STAGING_MODE=1`, so the mismatch was never caught. A premature DNS flip then pointed production at the empty, wrong-identity cluster — every existing agent/user cert would have been rejected. Root cause: an identity *inferred from an address* instead of *read from source*.

## Code-intelligence workflow

Understand structure BEFORE reading whole files; that makes Rule #1 fast.

- **Grep + Glob + targeted Read** (default — always available)
  - `Grep` 2-3 patterns to find usages, callers, references.
  - `Glob` for related files by naming convention (`*_test.go`, `*.proto`, `*/fixtures/*`).
  - `Read` with offset/limit — read the relevant section only.
  - Target: structural understanding in <5 tool calls before any Edit.
- Treat `e_imports.go` and `webassets_*.go` as the boundary between OSS and Enterprise/built-UI code paths.
- For protos: `proto/` and `api/proto/` are the source of truth; `gen/` and `api/gen/` are derived — never hand-edit derived files.

## Anti-pattern quick reference

| Symptom while working | Rule violated | Correct action |
|---|---|---|
| Editing a file not Read in this session | #1 | Read first, grep references, then Edit |
| Bug fix that also tweaks format/quote/import order | #3 | Touch only lines tied to the fix; restore the rest |
| Silently assuming format/scope/page-size, then implementing | #4 | State the assumption; offer 2-3 interpretations; ask |
| Strategy pattern / abstract base / config object for one use case | #5 | Direct function/method; refactor when a 2nd use case appears |
| `try/except` (or `if err != nil { return }` swallow) for impossible cases | #5 | Remove; only catch errors that can actually occur |
| Reporting "done" without running test/lint/build | #7 | Run verify, paste output, then report done |
| Writing "oh wait"/"actually"/"let me reconsider" a third time in one turn | #8 | Stop reasoning, go read more code |
| Assuming from memory/chat history without verifying current code | #9 | Trust codebase first; verify live/memory second |
| Letting `README.md`/`docs/`/`rfd/` lag behind a change | #9 | Update doc in the same commit/PR |
| Reading 20-30 files solo for a broad survey | #10 | Dispatch an `Explore`/`general-purpose` sub-agent |
| Launching sub-agents sequentially for independent work | #10 | Multiple `Agent` calls in one message → parallel |
| "Based on your findings, implement the fix" to a sub-agent | #10 | Sub-agent returns findings; main agent synthesizes and edits |
| Copying an address/DNS name into an identity field (cluster_name, CA pin, host UUID) | #12 | Read the real identity from source (`tctl status`, console); cite it; label identity vs address |
| Inferring "X is probably the same as Y" for an identifier, then baking it in | #12 | Verify the value from the live authoritative source before writing it anywhere |
