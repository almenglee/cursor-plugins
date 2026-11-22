# 08. Applying the Pyramid to Go (& Other Languages)

> **Core Insight:** *Lauren Tan's talk used Electron and TypeScript as her examples, but the Inverse Pyramid of Trust has nothing to do with JavaScript. In fact, **Go is one of the strongest languages on earth for this philosophy**—its compiler natively enforces architectural boundaries that TypeScript needs extra plugins to approximate, and its static-analysis ecosystem (`golangci-lint`, `go/analysis`, `go vet`, `-race`, native fuzzing, `benchstat`) maps 1-to-1 onto every layer of the Pyramid.*

---

## 1. Clearing Up the Myth: "There Is No Lint in Go"

Many Go developers feel confused when they hear talks centered on ESLint and custom lint rules, because Go culture famously avoids formatting wars:
- **`gofmt`** settled code formatting on day one, so Go never needed Prettier or 200 stylistic ESLint rules.
- **`go vet`** is intentionally conservative—the Go team only includes checks with near-zero false positives.

That creates the illusion that Go has "no lint." **The reality is the exact opposite.** Go ships its entire parser, AST, type-checker, and static-analysis framework in the standard library and `x/tools` (`go/parser`, `go/ast`, `go/types`, `go/packages`, `golang.org/x/tools/go/analysis`). On top of that sits **`golangci-lint`**, which aggregates 100+ semantic, architectural, security, and suppression linters into one blazing-fast binary.

Here is the direct 1-to-1 translation from Lauren Tan's TypeScript/Electron stack to Go:

| Concept in Lauren Tan's Talk (TS / Electron) | Direct Go Equivalent |
|---|---|
| `dependency-cruiser` / `no-restricted-imports` (`electron-main` vs `electron-renderer`) | **Go `internal/` packages** (compiler-enforced!) + **`golangci-lint` `depguard`** |
| Banning `useEffect` / `setTimeout` in product code | **`golangci-lint` `forbidigo`** (bans calls with custom fix messages!) |
| Branded types (`string & { readonly __brand: "UserId" }`) | **Go defined types** (`type UserID string` — native & zero-cost!) |
| Unforgeable validated tokens (`TenantContext`) | **Unexported struct fields** + single constructor in `internal/auth` |
| Exhaustive discriminated unions (`const _exhaustive: never`) | **Sealed interfaces** + **`exhaustive`** linter |
| Banning `@ts-ignore` / lazy `eslint-disable` (`comment-sicko`) | **`nolintlint`** (requires named linter + explanation on every `//nolint`) |
| Dead-code pruning ("gardening") | **`deadcode ./...`** (`golang.org/x/tools/cmd/deadcode`) |
| Runtime profiling & verification (CDP, traces, Bombadil, benchmark-checklist) | **`go test -race`**, **native `go test -fuzz`**, **`httptest` + `t.TempDir()`**, **`pprof`/`trace`**, **`benchstat`** |

---

## 2. Layer 1 in Go: Compiler-Enforced Architecture

Go gives you several Layer 1 superpowers right out of the box—no third-party tools required.

### 2.1 `internal/` Packages = Native Boundary Walls

In TypeScript, preventing `routes/` from importing low-level internals requires bundler/lint configuration. In Go, **the compiler itself enforces `internal/` visibility**: a package under `a/b/internal/c` can *only* be imported by code rooted at `a/b`.

Use a clean solo layout across CLI, backend, and web-API projects:

```text
my-go-app/
├── cmd/
│   ├── server/main.go          # Thin HTTP wiring only (wires deps, starts server)
│   └── cli/main.go             # Thin CLI shell (parses flags, renders text/--format json)
├── internal/
│   ├── domain/                 # Pure business types, state machines, rules (ZERO net/http, database/sql, os)
│   ├── auth/                   # Sole owner of TenantContext construction
│   ├── repo/                   # Sole owner of database/sql / sqlc queries
│   ├── httpapi/                # Boundary parsing (JSON -> domain types), status mapping
│   └── cliapp/                 # CLI command handlers calling domain + repo
├── pkg/contract/               # Optional shared wire DTOs / OpenAPI-generated types
├── scripts/check               # Single fast gate run locally & in CI
└── AGENTS.md                   # Short repo handbook + Rule → Enforcer table
```

### 2.2 Zero-Cost Semantic Branding & Unforgeable Capabilities

Go doesn't need TypeScript's `__brand` trick. Distinct defined types cannot be accidentally swapped by an agent:

```go
package domain

type UserID string
type OrgID  string

// Won't compile if an agent swaps (orgID, userID)!
func AssignOwner(orgID OrgID, userID UserID) error { ... }
```

And to make tenant authorization **impossible to bypass** (the Level 1 example from [Article 01](./01-inverse-pyramid-of-trust.md)), hide fields in an unexported struct inside `internal/auth`:

```go
package auth

import "my-go-app/internal/domain"

// Unexported field: NO package outside internal/auth can construct a valid TenantContext!
type TenantContext struct {
    orgID  domain.OrgID
    userID domain.UserID
}

func (t TenantContext) OrgID() domain.OrgID { return t.orgID }

// Only verified middleware/token parsing can produce a TenantContext.
func VerifyToken(rawToken string) (TenantContext, error) { ... }
```

Now every function in `internal/repo` accepts `ctx context.Context, tenant auth.TenantContext, ...`. An agent opening `internal/httpapi/orders.go` **cannot even compile** a database call without passing a real `auth.TenantContext` obtained from auth middleware.

### 2.3 Boundary Discipline & Sealed Sum Types in Go

- **Boundary parsing:** Parse `http.Request` JSON, CLI flags, and env vars *once* in `internal/httpapi` or `internal/cliapp` into validated `domain` structs. Inside `internal/domain`, trust the types and write pure functions with zero framework imports.
- **Sealed sum types (preventing invalid state bags):**
  ```go
  type JobState interface{ isJobState() }

  type JobPending   struct{ EnqueuedAt time.Time }
  type JobRunning   struct{ WorkerID string; StartedAt time.Time }
  type JobCompleted struct{ FinishedAt time.Time; Rows int }
  type JobFailed    struct{ ErrCode string; Retryable bool }

  func (JobPending) isJobState()   {}
  func (JobRunning) isJobState()   {}
  func (JobCompleted) isJobState() {}
  func (JobFailed) isJobState()    {}
  ```
- **Schema-first codegen:** Use **`sqlc`** (SQL → type-safe Go queries) and **`oapi-codegen`** or **Protobuf** so Go structs are derived directly from authoritative schemas instead of hand-synced copies.

---

## 3. Layer 2 in Go: Static Analysis & Agent-Directed Lint

Here are the three progressive tiers of static analysis in Go—from zero-dependency tests to custom compiler analyzers.

### Tier A: `.golangci.yml` Configured *for* AI Agents

Install `golangci-lint` and configure it so **every banned import or footgun prints the exact replacement file/function** (the `poteto` principle):

```yaml
version: "2"

linters:
  enable:
    - depguard        # Cross-layer import walls (Dune's electron-main/renderer rule in Go)
    - forbidigo       # Ban local footguns (Dune's useEffect/setTimeout ban in Go)
    - exhaustive      # Require handling every enum constant in switch statements
    - errcheck        # Never swallow returned errors
    - errorlint       # Enforce errors.Is / errors.As / %w wrapping
    - noctx           # Require context.Context on HTTP/SQL calls
    - bodyclose       # Catch unclosed HTTP response bodies
    - sqlclosecheck   # Catch unclosed sql.Rows / sql.Stmt
    - rowserrcheck    # Catch unchecked rows.Err() after DB iteration
    - nolintlint      # Go's Comment Sicko: force named rule + explanation on every //nolint
    - staticcheck     # Deep semantic bug detection
    - govet           # Standard vet + shadow checking

  settings:
    depguard:
      rules:
        domain-is-pure:
          files: ["**/internal/domain/**/*.go"]
          deny:
            - pkg: "database/sql"
              desc: "internal/domain must stay pure. Put DB access in internal/repo/ (see internal/repo/users.go)."
            - pkg: "net/http"
              desc: "internal/domain must stay transport-agnostic. Map HTTP concerns in internal/httpapi/."
        routes-use-repos:
          files: ["**/internal/httpapi/**/*.go"]
          deny:
            - pkg: "database/sql"
              desc: "Do not query database/sql from HTTP handlers. Call a TenantContext-scoped repo in internal/repo/."

    forbidigo:
      forbid:
        - pattern: '^time\.Sleep$'
          msg: "Do not use time.Sleep in app/test code; use context timers, channels, or a clock interface (internal/clock)."
        - pattern: '^fmt\.Print.*$'
          msg: "Do not use fmt.Print* for logging; use structured log/slog via the injected logger."
        - pattern: '^http\.DefaultClient$'
          msg: "http.DefaultClient has no timeout. Use internal/httpclient.New() with explicit timeouts."

    nolintlint:
      require-explanation: true   # Bans bare `//nolint` excuses
      require-specific: true      # Must name exact linter, e.g. `//nolint:forbidigo // CLI renderer`
      allow-unused: false         # Fails if a stale //nolint no longer suppresses anything
```

Look how closely `depguard` + `forbidigo` + `nolintlint` replicate Lauren Tan's entire static-analysis talk—often with *less* boilerplate than ESLint!

### Tier B: Zero-Dependency Architectural Test (`arch_test.go`)

Don't even want to configure an external linter first? Put a 35-line standard Go test in your repo root so plain **`go test ./...`** enforces your dependency graph automatically:

```go
package app_test

import (
    "strings"
    "testing"

    "golang.org/x/tools/go/packages"
)

func TestArchitectureBoundaries(t *testing.T) {
    pkgs, err := packages.Load(&packages.Config{Mode: packages.NeedName | packages.NeedImports}, "./internal/...")
    if err != nil {
        t.Fatalf("load packages: %v", err)
    }

    forbidden := map[string][]string{
        "/internal/domain":  {"database/sql", "net/http", "/internal/repo", "/internal/httpapi"},
        "/internal/httpapi": {"database/sql"},
    }

    for _, pkg := range pkgs {
        for layer, banned := range forbidden {
            if !strings.Contains(pkg.PkgPath, layer) {
                continue
            }
            for imp := range pkg.Imports {
                for _, rule := range banned {
                    if imp == rule || strings.Contains(imp, rule) {
                        t.Errorf("ARCH VIOLATION: %s must not import %s (fix: keep %s decoupled; see AGENTS.md)",
                            pkg.PkgPath, imp, layer)
                    }
                }
            }
        }
    }
}
```

Every AI harness already runs `go test ./...`. The instant an agent breaks a layer boundary, standard `go test` fails with your custom fix message!

### Tier C: Custom `go/analysis` Analyzers & Dead-Code Gardening

- **Custom type-aware lint rule:** When a rule requires type inspection (e.g., *"every exported repo method must accept `auth.TenantContext` as its second parameter"*), write a ~50-line `analysis.Analyzer` using `golang.org/x/tools/go/analysis` and run it directly via `go vet -vettool=./bin/mycheck ./...`.
- **Automated garden weeding:** Run `go run golang.org/x/tools/cmd/deadcode@latest ./...` periodically to list unreachable functions and delete dead code before agents copy it.

---

## 4. Layer 3 in Go: Built-In Verification Superpowers

Go makes Layer 3 (*Prove It Works*, *Explain the Number*, and property-based testing) unusually frictionless:

| Verification Need | Go Native Tool | Why It Shines With Agents |
|---|---|---|
| **Concurrency & race bugs** | `go test -race ./...` | Built-in ThreadSanitizer catches shared-state bugs deterministically—zero setup. |
| **Property-based / edge-case testing** | `go test -fuzz=FuzzParse -fuzztime=30s ./internal/parser` | Native coverage-guided fuzzer discovers panics, round-trip mismatches, and parser crashes tests never wrote. |
| **Isolated runtime verification** | `t.TempDir()` + `httptest.NewServer` + `testcontainers-go` | Every test gets a disposable temp directory & real HTTP/DB instance—perfect for *Separate Before Serializing Shared State*. |
| **Statistical benchmarks (*Explain the Number*)** | `go test -bench=. -benchmem -count=10` + `benchstat old.txt new.txt` | Computes medians, variance, and statistical significance (`p < 0.05` vs `~` noise)—prevents fake speedups automatically! |
| **CPU / heap / goroutine forensics** | `go test -cpuprofile=cpu.out` + `go tool pprof -top cpu.out` | Pure CLI output agents can read directly—eliminates being the "meat proxy" for profilers. |
| **Reusable Control CLI (`Build the Lever`)** | `go build -o bin/verify-ctl ./cmd/verify-ctl` | Compiles in <1s to a single zero-dependency binary with `--format json` and `--dry-run` that works identically in Cursor, Claude Code, Antigravity, and Codex. |

### Your Single Fast Gate (`scripts/check`)

Put one executable script in every Go repo so every harness runs the exact same bar locally and in CI:

```bash
#!/usr/bin/env bash
set -euo pipefail

go vet ./...
golangci-lint run ./...
go test -race -count=1 ./...
```

---

## 5. Cross-Language Rosetta Stone

Every principle in this wiki translates cleanly across languages—only the tool names change:

| Pyramid Layer | Go | TypeScript / Node | Python | Rust |
|---|---|---|---|---|
| **Layer 1: Boundaries** | `internal/` packages + unexported fields | Directory layout + `package.json` `exports` | Package structure + `_internal` | Cargo workspace crates (`pub(crate)`) |
| **Layer 1: Illegal States** | Defined types (`type UserID string`), sealed interfaces | Discriminated unions, branded types, `NonEmpty<T>` | `NewType`, `Literal` unions, `frozen=True` dataclasses | Enums with data, newtype structs (`struct UserId(String)`) |
| **Layer 1: Schema Source** | `sqlc`, `oapi-codegen`, Protobuf | `zod` (`z.infer`), OpenAPI, Prisma | `pydantic` v2, SQLAlchemy/Alembic | `serde` + `schemars`, `sqlx` compile-time queries |
| **Layer 2: Import / Footgun Lint** | `golangci-lint` (`depguard`, `forbidigo`, `nolintlint`) + `arch_test.go` | ESLint (`no-restricted-imports`, `no-restricted-syntax`), `dependency-cruiser`, Biome | `ruff` (`TID251`, `S`, `B`), `import-linter`, `tach`, `pyright --strict` | `clippy` (`#![deny(clippy::unwrap_used)]`), Cargo crate graph |
| **Layer 3: Fuzz / Property / Bench** | `go test -race`, `go test -fuzz`, `benchstat`, `pprof` | `fast-check`, Playwright/CDP, Bombadil, Vitest | `hypothesis`, `pytest`, Playwright | `proptest`, `cargo-fuzz`, `criterion`, `miri`, Kani |

---

**Previous:** ← [07. From Line Cook to Head Chef](./07-human-in-the-loop-delegation-and-factories.md) | **Home:** [Wiki Index](./README.md)
