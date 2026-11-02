# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project

Satchel is a privacy-first, local-first clipping vault: typed clips
(articles, places, movies, quotes…) with instant search, optional
end-to-end encrypted sync (self-hosted or Satchel Cloud), and MCP access
for AI clients. It is not a note-taking app.

Status: planning. Read these before starting work, and keep them up to
date when a decision changes:

- `docs/product-context.md` — what Satchel is and isn't, principles
- `docs/architecture.md` — components, storage, search, sync, MCP
- `docs/security.md` — threat model and encryption design
- `docs/modules.md` — the content-type module system
- `docs/roadmap.md` — phases and open questions

## Priorities

When trade-offs conflict, in this order:

1. **Privacy and security.** The server never reads vault contents.
   Nothing leaves a device without an explicit opt-in. No custom crypto.
2. **Offline first.** Every feature works against the local vault; the
   network is never on the capture, open or search path.
3. **Performance.** Search and filtering respond within a frame on a
   100k-item vault. Don't block the UI thread.
4. **Easy, automated backup** and a way out (Markdown export).

## Planned stack

- Rust workspace (`crates/`): core, extract, modules, crypto, sync,
  server, desktop (GPUI + GPUI Kit), ffi (UniFFI), mcp.
- pnpm workspace (`apps/`, `packages/`): Expo mobile app, WXT browser
  extension, generated TypeScript types.
- Rust: `cargo fmt`, `cargo clippy --all-targets -- -D warnings`,
  `cargo test`. TypeScript: pnpm only (not npm or yarn), oxlint, oxfmt,
  vitest.

## Git workflow

### Branches and merging

- `main` has linear history and stays releasable.
- Do work on a short-lived branch and open a pull request. Do not push
  directly to `main`.
- PRs are **squash merged**. The PR title becomes the commit subject on
  `main` and the PR description becomes its body, so both must follow the
  commit conventions below.
- Rebase on `main` to pick up changes; don't merge `main` into a branch.
- Delete branches after they merge.
- Never force-push `main`. Force-push a feature branch only with
  `--force-with-lease`, and only your own branch.

### Commits

Use [Conventional Commits](https://www.conventionalcommits.org/) with a
**required scope**:

```text
<type>(<scope>): <description>
```

- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`,
  `build`, `ci`, `chore`, `revert`.
- Scope is required, lowercase, and names the area changed: a crate or
  app (`core`, `sync`, `crypto`, `server`, `desktop`, `mobile`,
  `extension`, `mcp`), a module (`place`, `movie`), or a cross-cutting
  area (`deps`, `docs`, `ci`, `repo`).
- Description: imperative mood (`add`, not `added`/`adds`), lowercase
  first word unless a proper noun, no trailing period, subject line
  ≤ 72 characters (aim for ≤ 50).
- Breaking changes: `!` after the scope (`feat(sync)!: ...`) plus a
  `BREAKING CHANGE:` footer.
- Body: explain why and what, not how; wrap at 72 characters.
- Do not add `Co-Authored-By` lines to commits or PRs.

Examples:

```text
feat(core): add fuzzy header index
fix(sync): resume pull after interrupted snapshot
perf(desktop): virtualize result grid
chore(deps): update gpui-component
docs(security): describe device revocation
```

### Grouping changes

- One logical change per commit: a reviewer should be able to describe
  it in one sentence.
- Before committing, review staged and unstaged diffs and split them into
  logical groups. Keep unrelated fixes, features, refactors, formatting,
  dependency updates and docs in separate commits, even when asked to
  "commit everything", unless explicitly told to combine them.
- Each commit should build and pass tests on its own.
- Keep PRs focused on one change. Split large work into a series of PRs
  rather than one large one.

### Pull requests

- Title: must follow the same Conventional Commits rules as commits,
  including the **required scope** (`<type>(<scope>): <description>`),
  because it becomes the commit subject on `main` when squash merged.
  Update the title if the PR's scope changes during review.
- Description: why the change is needed, what changed, and anything a
  reviewer should look at closely. Follow-up work goes in a short list
  at the end.
- Include a test plan section only when asked.
- Run formatting, lint, type checks and tests locally before opening or
  updating a PR.

### General

- Never commit secrets, `.env` files, vault databases or test vaults
  containing real data.
- Don't commit generated build output; commit lockfiles.
- Don't rewrite published history other than your own unmerged branch.
