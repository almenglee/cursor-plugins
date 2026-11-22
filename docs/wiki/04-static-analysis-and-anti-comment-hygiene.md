# 04. Static Analysis, Agent-Directed Lint & Why Comments Lie

> **Core Philosophy:** *"Comments go stale, humans stop reading comment slop, and comments give agents an escape hatch to justify skipping the right fix. Worse, comments cannot fail a build. Replace prose warnings with lint rules whose error messages tell the agent how to fix the root cause."*

---

## 1. Why Comments Become Slop Factories in Agent Codebases

One of the most provocative data points in Lauren Tan's *"All Code Is Slop (Except Mine)"* talk was a single bar chart: her team cut code comments in Grok Bot from **65,824** (July 31) down to **1,227**—eliminating roughly **99% of comments**.

That sounds extreme until you examine **how AI agents actually use comments**. Unlike humans, agents routinely use comments for four destructive habits:

1. **Rationalizing workarounds ("Permission Slips"):** When the clean architectural fix touches 6 files and a hacky `setTimeout` touches 1 file, an agent writes the hack and adds a convincing 4-line comment explaining why it was "necessary for now." The next agent reads that prose as authoritative architectural wisdom and copies the hack everywhere (how 1 timer became **183 timers** in five weeks).
2. **Narrating the obvious:** `// Increment counter`, `// Fetch user from database`, `// --- Helpers ---`. Pure token noise that clutters context windows.
3. **Unenforceable warnings:** `// IMPORTANT: do not remove this call or sync breaks!` or `// NOTE: keep this list in sync with routes.ts`. Because nothing fails when an agent ignores a comment, these warnings rot silently while giving engineers a false sense of safety.
4. **Suppressing real bugs:** `// eslint-disable-next-line react-hooks/exhaustive-deps` or `// @ts-ignore fine for now`—silencing the very deterministic guardrails meant to protect the codebase.

### The Narrow Keep-List (When a Comment *Is* Legitimate)

In `pstack`'s adversarial comment reviewer (`comment-sicko`), every comment dies **unless** it qualifies under one of five strict exceptions:

| Exception | Why It Survives |
|---|---|
| **Legal / License headers** | External compliance requirement. |
| **External quirk you cannot reshape** | Non-obvious behavior forced by a third-party vendor, OS, browser, or protocol you don't own (proven true on a live path today—surprises in *your own code* get renamed, typed, or refactored instead!). |
| **Public API doc contracts** | Structured doc comments defining a public library/SDK contract. |
| **Issue / RFC links for unexpressible constraints** | A concrete link explaining an external constraint code truly cannot express. |
| **Faulty/style-only formatter ignore** | e.g., `// prettier-ignore` on an alignment matrix. Correctness or safety lint suppressions (`@ts-ignore`, type/bug rules) are **never** allowed to linger. |

> **The Diagnostic Habit:** Treat long justifying comments (`IMPORTANT`, `do not remove`, `too risky`, `fine for now`) as **scent pointing to a bug or missing static check**. Whenever a comment claims a constraint (*"must match schema X"*), ask: *can a type, test, or lint rule enforce this instead?* If yes, encode the check and delete the comment.

---

## 2. Writing Lint Rules *for Agents* (Not Just Humans)

Most lint configs were designed for human developers who already know the codebase. When an AI agent hits a generic lint error, it often thrashes or adds an `eslint-disable` suppression.

**Agent-directed static analysis follows one simple principle:** *every lint error message is a high-priority, un-ignorable prompt injected at the exact moment of mistake.*

### Compare Weak vs. Agent-Directed Lint Messages

```js
// WEAK: Agent knows WHAT failed, not WHAT to use instead -> may hack around it
"no-restricted-imports": ["error", { patterns: ["**/io/db*"] }]

// AGENT-DIRECTED: Tells the agent the exact replacement pattern & reference file
"no-restricted-imports": ["error", {
  paths: [{
    name: "../io/db",
    message:
      "Do not import raw db in routes/. Call a typed repository in src/io/repos/ " +
      "(see src/io/repos/users.ts) with a verified TenantContext."
  }]
}]
```

The same principle applies to banning local footguns (like `useEffect`, raw `setTimeout`, or `console.log`):

```js
// Example: Banning raw setTimeout in UI components (the Dune lesson)
"no-restricted-globals": ["error", {
  name: "setTimeout",
  message:
    "Do not use raw setTimeout in web/features/ — imperative timers cause GC churn and race conditions. " +
    "Use useDebouncedValue() or scheduleTask() from web/primitives/timers.ts, or fix the underlying state transition."
}]
```

Watch what happens inside an agent session:
1. Agent writes `setTimeout(...)` to patch a flicker.
2. Agent runs `./scripts/check` (takes 2 seconds).
3. Linter fails and tells the agent **why** it's banned and **which file** holds the supported alternative.
4. Agent reads `web/primitives/timers.ts` and fixes the root cause in the very same turn—you never see the bad code in review.

---

## 3. Enforcing Architectural Boundaries Mechanically

Directory structure only works as a guardrail if cross-boundary imports fail the build—including **type-only imports** when processes/runtimes must stay cleanly separated via `shared/`.

| Ecosystem | Best-in-Class Boundary Tools | Example Rule |
|---|---|---|
| **TypeScript / JS** | `dependency-cruiser`, `eslint-plugin-boundaries`, Biome / ESLint `no-restricted-imports`, Node `package.json` `exports` | `domain/**` cannot import `io/**`, `cli/**`, or `server/**`; `web/**` cannot import `server/**`. |
| **Python** | `import-linter`, `tach`, Ruff `TID251` (`banned-api`) | `layers` contract: `cli` / `api` → `domain` → `shared`; `domain` forbidden from importing `sqlalchemy` or `fastapi`. |
| **Go** | `internal/` package visibility + `golangci-lint` (`depguard`) | `depguard` blocks `database/sql` inside `internal/http/...`. |
| **Rust** | Cargo workspace crates (`shared`, `domain`, `io`, `cli`, `server`) | `domain/Cargo.toml` literally does not list `sqlx` or `axum` as dependencies—impossible to import! |

Notice how **Rust workspaces** and **Go `internal/`** are Layer 1 perfection: if `domain/Cargo.toml` doesn't depend on the database crate, no agent in the universe can accidentally import it.

---

## 4. The Ratchet Pattern: Hardening Messy Legacy Repos Without Stopping Work

One of the biggest excuses for skipping static analysis is: *"I already have 50 `any` casts and 12 cross-layer imports in my project—I can't turn on the lint rule without spending a week fixing old code!"*

Lauren's `/correct` workflow solves this cleanly with **ratcheting**:
> *"If the pattern is already common, fail only when a change adds more."*

Here is a complete, 35-line language-agnostic ratchet script pattern you can drop into any repo (`scripts/check-ratchet.mjs`):

```js
#!/usr/bin/env node
import { execSync } from "node:child_process";
import { readFileSync, writeFileSync } from "node:fs";

const BASELINE_PATH = "scripts/ratchet-baseline.json";
const baseline = JSON.parse(readFileSync(BASELINE_PATH, "utf8"));

const CHECKS = {
  // Pattern name -> ripgrep command counting matches in src/
  ts_ignore_or_any: `rg -n "@ts-ignore|@ts-expect-error|\\bas any\\b" src/ || true`,
  raw_console_log:  `rg -n "\\bconsole\\.log\\(" src/domain src/server || true`,
  raw_set_timeout:  `rg -n "\\bsetTimeout\\(" src/web/features || true`,
};

let failed = false;
const nextBaseline = { ...baseline };

for (const [name, cmd] of Object.entries(CHECKS)) {
  const lines = execSync(cmd, { encoding: "utf8" }).trim().split("\n").filter(Boolean);
  const count = lines.length;
  const max = baseline[name] ?? 0;

  if (count > max) {
    console.error(`FAIL [${name}]: found ${count}, max allowed is ${max}. Remove the new occurrences:\n${lines.slice(0, 10).join("\n")}`);
    failed = true;
  } else if (count < max) {
    console.log(`IMPROVED [${name}]: ${max} -> ${count}. Run with --update to lock in the lower ceiling.`);
    nextBaseline[name] = count;
  }
}

if (process.argv.includes("--update") && !failed) {
  writeFileSync(BASELINE_PATH, JSON.stringify(nextBaseline, null, 2) + "\n");
}
process.exit(failed ? 1 : 0);
```

**Why ratcheting works so well:**
- Zero upfront refactor tax—you can turn it on in 5 minutes today.
- **Zero new slop can enter** from this commit forward.
- Every time an agent cleans up an area, the ceiling drops until it hits `0`, at which point you convert it to a permanent hard lint rule.

---

## 5. Custom Repo-Invariant Scripts & Time-Boxed Exceptions

Generic linters don't know your repository's domain invariants. Borrow the exact pattern used in this repository's own [`scripts/check-pstack-docs.mjs`](../../scripts/check-pstack-docs.mjs) (which parses English counts, links, slash commands, and model names across docs and fails CI on drift):

Write small, fast (50–150 line) invariant checks in `scripts/check-invariants.*`:

1. **Expired exception guard:** Require every inline suppression to carry an expiry date (`// lint-exception(expires:2026-12-31): <reason>`) and fail CI automatically once that date passes.
2. **CLI contract guard:** Verify every command registered in `src/cli/commands/` is documented in `--help` and has a contract test.
3. **Doc/Feature-map drift guard:** Verify every file path or command referenced in `AGENTS.md` and `docs/verification/` actually exists on disk.
4. **Prove every new check on a real failure:** Never merge a new static rule until you have run it against an actual past buggy commit or snippet and watched it fail with a clear fix message.

---

## 6. Observe-Only Background Sweepers (Finding 1 Root Cause Instead of 12 Symptoms)

One of the subtlest gems from Lauren Tan's Matt Pocock interview is her **observe-only background sweeper**:

- She runs a background agent that scans the codebase for React anti-patterns—**but it is strictly forbidden from editing code.**
- Instead, it appends every finding to a single log document.
- Every few days, she spends 10 minutes reading the accumulated log and frequently discovers that **a dozen seemingly separate issues are all symptoms of one missing primitive or architectural flaw**.

If you unleash agents to "fix every anti-pattern they find" immediately, they apply 12 piecemeal band-aids. By separating **observation** from **structural remediation**, you fix the architecture once and eliminate all 12 issues at the root.

---

**Previous:** ← [03. Types, Boundaries & Domain Modeling](./03-types-boundaries-and-domain-modeling.md) | **Next:** [05. Verification as the New Code Review](./05-verification-as-code-review.md) →
