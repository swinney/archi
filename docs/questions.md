# Open questions

The Ralph loop appends here when it hits a decision the specs don't cover, then
stops without committing code (see PROMPT.md "Stop conditions"). Answer a question
by resolving it in the relevant spec or a decision record, then remove it from this
list. An empty list below means nothing is currently blocked.

<!-- The loop appends entries below this line. -->

## Task 2.1 (fix-issue-407-fence-delimiter-backtick-runs) — PR creation blocked by token scope

**Status: BLOCKED — GH_TOKEN lacks pull_request write scope on `fasrc/archi`.**

All implementation work for `fix/issue-407-fence-delimiter-backtick-runs` is complete
and green: `bash scripts/gate.sh` exits 0 (3652 passed, diff-cover 100% on
`src/data_manager/collectors/processing.py`), `git status` is clean, `git diff
origin/dev --stat` touches only `src/data_manager/collectors/processing.py`,
`tests/unit/test_html_to_markdown_processor.py`, and this change's
`openspec/changes/fix-issue-407-fence-delimiter-backtick-runs/` files, and the
design.md "Verification" `markdown-it-py` script prints `PASS`. The branch is pushed
to `swinney/archi` (fork; same pattern as the #335 precedent below). However:

- `git push -u origin fix/issue-407-fence-delimiter-backtick-runs` fails: the
  `swinney` account has no push access to `fasrc/archi` (403, reproduced twice
  including with `GIT_CURL_VERBOSE`; `gh api repos/fasrc/archi --jq '.permissions'`
  for the same token reports `push: true`, so the token's reported metadata and the
  actual git ACL disagree).
- `gh pr create --repo fasrc/archi --base dev --head swinney:fix/issue-407-fence-delimiter-backtick-runs ...`
  fails: `Resource not accessible by personal access token (createPullRequest)`.

**Resolution needed:** a human should open the PR from
`swinney:fix/issue-407-fence-delimiter-backtick-runs` → `fasrc/archi:dev`, or provide
a token with PR-write access to `fasrc/archi`. The PR body should include:
- `Closes #407`
- **What**: `_ArchiMarkdownConverter(MarkdownConverter)` overrides `convert_pre` to
  size the fence to `max(3, longest_backtick_run + 1)` instead of a fixed three
  backticks, via the seam `_markdownify()` called from `_worker()`
  (`processing.py:411`).
- **Measured outputs**: `<p><code>a<br>```<br># heading</code></p>` before
  `` '```\na\n```\n# heading\n```' `` / after `` '````\na\n```\n# heading\n````' ``
  (design.md D5 has the full table).
- **Corpus**: 0 of 25 promoted and 0 of 145 native `<pre>` in the 60-page sample carry
  a run of three or more backticks — no persisted text changes for the sample.
- **Verification**: the design.md `markdown-it-py` script prints `PASS`.
- **Related**: PR #405 (Codex thread
  https://github.com/fasrc/archi/pull/405#discussion_r3912257990); PR #414 for #406
  touches the same file (whichever merges second rebases); #410 adds `convert_list`
  to `_ArchiMarkdownConverter`.
- **No re-ingest and no redeploy in this PR.**

## Task 5.2 — "Run before/after benchmark; record recall/precision deltas"

**Status: BLOCKED — requires live infrastructure not available to the loop.**

The first unchecked task in `tasks.md` is section 5.2, which asks to *run* the
benchmark before and after the title-aware retrieval change and record the
recall/precision deltas. This cannot be executed in the Ralph loop sandbox:

- `src/bin/service_benchmark.py` reads `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, and
  `HUGGING_FACE_HUB_TOKEN` via `read_secret(...)` at import time, and constructs a
  Postgres connection through `PostgresServiceFactory.from_env(...)`. None of these
  secrets are present (`.env` is empty; no `.secrets.env`).
- A real before/after run needs an **ingested corpus** whose documents have the
  title-only / filename-only keywords described in
  `src/bin/benchmark_query_sets/title_aware_query_set.json`. The harness blocks on
  the data-manager ingestion-status endpoint (`wait_for_ingestion_completion`)
  before scoring.
- The environment has **no container runtime** (neither `docker` nor `podman` on
  PATH) and **no reachable Postgres/pgvector** (port 5432 closed), so a deployment
  cannot be brought up to ingest the corpus or serve retrieval.
- Producing recall/precision numbers without that stack would mean fabricating
  results, which the spec ("Retrieval quality is benchmarked") and the loop's
  "report outcomes faithfully" rule forbid.

**Decision needed from a human operator:** run the benchmark on a real deployment
and record the deltas, OR clarify how the loop should satisfy 5.2 offline (e.g.
a fixture-backed, deterministic mini-corpus + an offline benchmark path that does
not require secrets or a container runtime). The two "baseline vs. new behavior"
configs should be the `title_header.enabled` / `title_weight` / `filename_boost`
knobs toggled off vs. on (see `add-title-aware-retrieval` design.md, Migration
Plan step 4). Until resolved, tasks 5.2–5.4 (which depend on the recorded
results) and the test/validation work in section 6 remain queued behind this.
