# Agents — kbexplorer-provider-rich-markdown

`@anokye-labs/kbexplorer-provider-rich-markdown` is a **loadable kbexplorer
provider**: it ingests one rich Markdown document into a `KBGraph` fragment. It
is pluggable infrastructure (same category as an iCal or dot-graph reader) and is
loaded by hosts through the `config.yaml` → `defineProvider()` seam. It depends
ONLY on `@anokye-labs/kbexplorer-core` (`v0.3.0`).

Keep the parsing/extraction **library** (`src/lib/`) pure — no filesystem, no
network, no LLM, identical input → byte-identical output. The only I/O boundary
is the thin provider wrapper (`src/providers/`), which reads source bytes then
delegates to the library.

## Stack

Plain ESM JavaScript, **no build step**. Tested with Node's built-in `node:test`
runner via `scripts/run-tests.js`. (This deliberately differs from
`kbexplorer-core`/`kbexplorer-search`, which are TypeScript + a bundler — the
extracted source here is already pure ESM JS and needs no transpile.)

## Build / Test

```bash
npm ci
npm test            # node --test
```

## Exports / Compatibility

Two entrypoints, both public API:

- `.` — the `defineProvider()` factory (default export) + `apiVersion` /
  `capabilities` + error types.
- `./lib` — the pure ingestion library.

Treat both as public: additive changes are safe; renames/removals are breaking
and must be coordinated with consumers (`kbexplorer-cli`, `kbexplorer-template`).

## Branch Protection

**Check, don't assume.** The rules below describe the *intended* policy, but a
live audit (anokye-labs/kbexplorer#105) found `kbexplorer-core`'s `main` has
**no branch-protection ruleset configured at all**, despite that repo's own
AGENTS.md previously stating otherwise — the same drift is plausible here.
Verify actual repo settings (`/rules/branches/main` via the API, or the repo's
Settings → Rules UI) before relying on any of the following as fact:

- **Pull request required** — no direct pushes to `main`.
- **Required status checks** (strict / up-to-date) — `pr-title`,
  `check-linked-issue`, `dependency-review`, `test`.
- **Conversation resolution required** before merge.
- Force pushes and branch deletion are blocked.

Never commit directly to `main`. Never force push.

## Issue-First Workflow

**Every pull request must trace back to a GitHub Issue.**

1. Create an Issue (with a native Issue **Type**: Epic / Feature / Task / Bug).
2. Create a branch and implement.
3. Open a PR that references the issue (e.g. `refs #12`) — closure is a
   separate, post-verification step (see below), not something a PR
   description should trigger automatically.
4. CI goes green → merge. Close the issue explicitly once the merged change has
   been verified, rather than relying on merge-time auto-closing.

Use the **GraphQL API** for issue types, sub-issues, and blocked-by
relationships (the REST API does not support them). Include
`GraphQL-Features: sub_issues` for sub-issue operations.

## GitHub & Work-Item Conventions

These conventions are tool-agnostic and shared across the org's repos — apply
them however you interact with GitHub (`gh` CLI, REST, GraphQL, or an MCP
server); no tool is preferred over another. GraphQL-level capability is
required specifically for sub-issues and blocked-by relationships, since the
REST API cannot express them.

- **`refs #N`, never `closes #N`** in commits/PRs — linking is not closing.
- **Verify before closing.** An issue closes only after its fix has been
  independently verified (tests pass, behavior checked) — never as an
  automatic side effect of a merge or a commit keyword.
- **Conventional Commits** (`type(scope): description`) for every commit.
- For **work-breakdown-structure mechanics** (typed Epic → Feature → Task
  hierarchies, sub-issues, blocked-by edges via GraphQL), see
  `kbexplorer-template`'s `.agents/skills/wbs-builder/` skill rather than
  reinventing the scaffolding here.

## Verification

Run the tests before handing back control. If you cannot fully verify a change,
say so explicitly and explain why — never claim "done" with a silent gap.
