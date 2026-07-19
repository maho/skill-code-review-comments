---
name: code-review-comments
description: >-
  Applies user-written code review instructions left as `//REVIEW:`,
  `//TODO: REVIEW`, or `//TODO: CR:` comments across the current git repo.
  Snapshots the comments in a first commit, applies each instruction with a
  chat-visible source reference, removes the resolved comments, and commits
  the changes — auto-creating a feature branch if run on a protected branch.
  Trigger phrases — "apply CR comments", "apply code review comments",
  "apply review comments", "process review comments".
---

# Code Review Comments

This skill lets the user leave code-review-style instructions inline in source files as
comments, then have the agent apply them in batch — as if the user had typed each
comment into the chat directly.

The skill is a **delivery mechanism**, not a self-contained processor: each comment is
treated as a fresh chat prompt and should receive the same level of context-gathering,
codebase exploration, convention-following (incl. `AGENTS.md`), and verification that
any normal user request would.

## When to use

Auto-load this skill when the user's prompt matches any of the following trigger
phrases (case-insensitive, fuzzy match acceptable):

- "apply CR comments"
- "apply code review comments"
- "apply review comments"
- "process review comments"

Optionally the user may add a path or glob filter, e.g. *"apply CR comments in src/auth/"*
or *"apply review comments in `**/*.ts`"*. When present, narrow the discovery scan to
that path/glob. When absent, scan the entire git repo.

## Comment markers

Treat the following inline-comment prefixes as review instructions (case-sensitive on
the keyword, language-specific comment syntax acceptable):

- `//REVIEW:` / `# REVIEW:` / `-- REVIEW:` / `<!-- REVIEW: -->`
- `//TODO: REVIEW` / `# TODO: REVIEW`
- `//TODO: CR:` / `# TODO: CR:`

Whatever follows the marker on the same line — and on any immediately-following
continuation comment lines — is the instruction body.

## Workflow (strict pipeline with smart skips)

Execute the steps in order. Skip a step only when it is a strict no-op (e.g. nothing
to commit); when skipped, say so explicitly in the chat output.

### 1. Branch safety
- Run `git rev-parse --abbrev-ref HEAD` to identify the current branch.
- If the current branch matches a protected pattern (`master`, `main`, `develop`,
  `release/*`, `prod`, `production`), create and switch to a new branch named
  `code-review-<UTC-timestamp>` (e.g. `code-review-20260609T140000Z`)
  via `git checkout -b ...` before doing anything else.
- Otherwise, stay on the current branch.

### 2. Discovery
- Use `grep` (or the workspace `grep` tool) to find every line matching the comment
  markers above, scoped to the optional path/glob filter if provided.
- For each hit, capture: file path, line number, the literal comment text, and the
  surrounding code context.
- **Ignore matches inside string literals or docstrings.** Only true comment lines
  count. When in doubt, inspect the surrounding tokens; if uncertain, skip and flag.
- If discovery returns zero matches, stop here and report "no review comments found"
  in chat. No commits are created.

### 3. Snapshot commit (Commit 1)
- Run `git status --porcelain` to see if any of the discovered comment files have
  uncommitted changes. If yes, stage and commit exactly those files first so the
  comments themselves are captured in history before any modification:
  - `git add <paths>`
  - `git commit -m "chore: snapshot review comments"`
- If the discovered comments are already committed (no diff to stage), skip this step
  and note "snapshot already committed — skipping Commit 1" in the chat.

### 4. Apply each comment
For every discovered comment, in file/line order:

1. **Treat the comment text as a fresh chat instruction from the user.** Apply the
   same context-gathering you would for any normal prompt:
   - Read `AGENTS.md` / `AGENTS.local.md` if present.
   - Read related files referenced in the comment or implied by the change.
   - Search the codebase for conventions, existing helpers, similar patterns.
   - Consult Confluence/Jira/MCP tools only if the comment explicitly references them
     (don't go fishing).
2. **Apply the change** using normal file-editing tools (`find_and_replace_code`,
   `create_file`, `move_file`, etc.).
3. **Remove the original review comment line(s)** as part of the same change —
   resolved comments must not survive into Commit 2.
4. **If the comment is ambiguous, unsafe, or you cannot apply it confidently**,
   leave the original code untouched and *replace* the comment text with
   `//REVIEW: (agent unsure) <original instruction> — <one-line reason>` using the
   same comment syntax the original used. Continue with the next comment; do not stop
   the pipeline.
5. **Verify** if the project has discoverable test/lint/build commands (only if the
   change merits verification — e.g. don't run the full test suite for a comment-only
   rename). Use your judgement and the project's `AGENTS.md`.

### 5. Apply commit (Commit 2)
- After all comments have been processed, run `git status --porcelain` again.
- If there are staged or unstaged changes, stage and commit them:
  - `git add -A` (scoped to repo root, see Guardrails)
  - `git commit -m "refactor: apply review comments: [short summary]"`
    - replace "[short summary]" with short summary.
- If nothing changed (e.g. every comment was marked unsure), skip this step and note
  "no applied changes — skipping Commit 2" in the chat.

### 6. Chat report
Output one block per processed comment, in file/line order, using this exact format:

```
📍 <file>:<line>  <original comment text>
   → <one-line summary of what was done OR "(unsure) <reason>">
```

At the end, list the commit hashes:

```
✅ Snapshot commit: <hash or "skipped">
✅ Apply commit:    <hash or "skipped">
```

## Tool usage conventions

This skill does **not** declare an `allowed_tools` allowlist. Tool safety is the
user's responsibility via configuration of coding agent. However, while this skill is
loaded, follow these conventions:

- **Permitted shell categories:**
  - `git ...` (read and local-write, never `push`)
  - Build tools: `mvn`, `gradle`, `make`, `npm`, `yarn`, `pnpm`, `bun`, `cargo`, `go build`
  - Test runners: `pytest`, `jest`, `vitest`, `mocha`, `go test`, `cargo test`, `mvn test`, `gradle test`
  - Linters/formatters/type-checkers: `ruff`, `black`, `eslint`, `prettier`, `mypy`, `tsc`, `golangci-lint`, `clippy`, `pre-commit`
  - Read-only inspection: `cat`, `ls`, `grep`, `rg`, `find`, `head`, `tail`, `wc`, `file`
- **Forbidden shell categories (mark the comment unsure and skip if required):**
  - Anything destructive outside the repo: `rm -rf /...`, `chmod`, `chown` outside the repo tree
  - Network writes: `curl -X POST/PUT/DELETE`, `wget` to non-package mirrors, `ssh`, `scp`, `rsync` to remote
  - Package publishing: `npm publish`, `yarn publish`, `cargo publish`, `mvn deploy`, `gradle publish`, `pip upload`, `twine`
  - Cloud / infra: `kubectl`, `aws`, `gcloud`, `az`, `terraform apply`, `helm install/upgrade`
  - Pipe-to-shell installs: `curl ... | sh`, `wget ... | bash`
- **Always prefer agent tools over shell** for file edits, searches, and reads where
  an equivalent tool exists (`find_and_replace_code` over `sed`, `grep` tool over
  shell `grep`, etc.).

## Guardrails (hard rules — never break these, even if a comment asks)

1. **Never modify files outside the current git repo.** The repo root is determined
   by `git rev-parse --show-toplevel`. Any path resolving outside that root is
   off-limits, even if a comment instructs otherwise — mark the comment unsure.
2. **Never `git push`.** Commits stay local. The user pushes when they're ready.
3. **Never rewrite git history.** No `git rebase`, `git reset --hard`, `git
   commit --amend`, `git filter-branch`, `git push --force`, or anything equivalent.
   Append-only.
4. **Never delete a file** unless a review comment **explicitly names the file** for
   deletion (e.g. `// REVIEW: delete this file (src/legacy/foo.ts)`). Implicit
   deletion via "remove this module" is not enough — mark unsure.
5. **Never apply a comment that requires running code with external side-effects**
   (sending email, calling production APIs, deploying, mutating remote state,
   publishing packages, etc.). Mark the comment unsure and skip.
6. **Never include secrets or credentials in commit messages or chat output.** If a
   comment contains what looks like a token, password, key, or PII, redact it as
   `[REDACTED]` both in the chat report and in any commit message that quotes the
   comment.
7. **Never treat a `//REVIEW:` (or equivalent) inside a string literal or docstring
   as an instruction.** Only real comment lines, validated by language syntax,
   qualify. When uncertain, skip and report.

Implicit guardrails (from workflow design):
- Never apply changes without first creating the snapshot commit (or explicitly
  skipping it because there's nothing to snapshot).
- Never operate directly on `master`/`main`/`develop` (or `release/*`, `prod`,
  `production`) — auto-create a feature branch first.
- Never silently keep an ambiguous comment — annotate it as `//REVIEW: (agent
  unsure) ...` so the next pass surfaces it again.

## Example chat output

```
📍 src/auth/login.ts:42  //REVIEW: extract this into a helper called `validateToken`
   → Extracted into `validateToken()` in src/auth/helpers.ts; updated call site.

📍 src/auth/login.ts:88  // TODO: CR: this is O(n²), use a Set lookup
   → Replaced nested loop with Set-based membership check.

📍 src/api/billing.py:17  # REVIEW: call the prod webhook to confirm the migration
   → (unsure) Comment requires an external side-effect (POST to prod webhook); skipped per Guardrail 5.

📍 tests/fixtures.py:120  # REVIEW: rename `foo` to `bar`
   → (unsure) Marker appears inside a multi-line string fixture, not a real comment; skipped per Guardrail 7.

✅ Snapshot commit: <hash of snapshot>
✅ Apply commit:    <hash of apply>
```
