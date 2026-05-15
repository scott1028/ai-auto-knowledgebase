# claude_schedule_prompt.md

Entry prompt for scheduled / remote Claude Code runs of the
`ai-auto-knowledgebase` repository. This file is the cold-start
brief: when the scheduler wakes an agent that does NOT have the
repo checked out locally, the agent reads rule files directly
from GitHub raw URLs and treats those URLs as its working tree.

## Step 1 — Fetch the rule files

Fetch the following three files first. They are the authoritative
source of every rule, workflow, and source list this run must
obey:

1. `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/README.md`
2. `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/mission_prompt.md`
3. `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/knowledge_website.md`

## Step 2 — Treat the raw URL root as `./`

For the duration of this run, the virtual working directory is:

```
https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/
```

Any relative path referenced inside the fetched files (e.g.
`./.knowledge/`, `./recent_updates.md`, `./CLAUDE.md`) MUST be
resolved against that URL prefix. Example:

- `./CLAUDE.md` →
  `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/CLAUDE.md`
- `./recent_updates.md` →
  `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/recent_updates.md`

If a rule refers to a local filesystem operation that does not
make sense over raw HTTP (e.g. writing to `.knowledge/`,
committing, pushing), surface the limitation in the final report
rather than attempting it — this scheduled run is read-mostly and
its primary output channel is Gmail (see Step 5).

## Step 3 — Parse, then fan-out fetch

After the three rule files are fully read and understood:

1. Extract the source URLs / feeds / topic lists they reference
   (e.g. entries under `knowledge_website.md`, candidate URLs
   inside `mission_prompt.md`).
2. Apply the dedup and topic-relevance filters described in
   `CLAUDE.md` BEFORE fetching any candidate URL.
3. Fetch only the URLs that survive both filters.
4. Process each surviving source per the ingestion workflow in
   `mission_prompt.md` and `CLAUDE.md`.

Do NOT fetch arbitrary URLs that are not derived from the rule
files. Do NOT skip the dedup / topic-filter steps just because
this is a remote run.

## Step 4 — HTTP retry policy

Every HTTP fetch in Steps 1 and 3 MUST be retried on failure:

- Minimum retries: **3 attempts** (i.e. up to 3 retries after the
  initial failure, for a total of 4 attempts).
- Backoff: fixed **5 seconds** between attempts.
- A failure is any of: non-2xx HTTP status, network/connection
  error, timeout, empty body when a body was expected.
- If all attempts fail for a rule file in Step 1, ABORT the run
  and emit a single Gmail draft titled
  `ai-auto-knowledgebase scheduled run — aborted (fetch failure)`
  with the failing URL and the last error in the body. Do not
  proceed with partial rules.
- If all attempts fail for a content URL in Step 3, SKIP that
  single URL, record the failure in the final report, and
  continue with the remaining URLs.

## Step 5 — Output via Gmail

The final output of this scheduled run is one or more Gmail
drafts, created via the Gmail MCP integration, following the
draft rules in `CLAUDE.md` — in particular:

- **One draft per knowledge file** (strict 1:1 mapping). Never
  bundle multiple knowledge files into a single draft.
- Each subject ends with the corresponding
  `YYYY-MM-DD_<slug>_<sourcehash>.md` filename after a ` - `
  separator.
- Run the dedup check against existing drafts before creating a
  new one; skip duplicates and report the skip.

If the run produces no new on-topic, non-duplicate knowledge,
emit a single short status draft summarising the dedup / skip /
off-topic counts instead — do not stay silent.

## Step 6 — Final report

At the end of the run, the agent's last visible message MUST
contain:

- Count of rule files fetched successfully.
- Count of candidate URLs evaluated, broken down by: written,
  dedup-skipped, off-topic-skipped, fetch-failed.
- For each written knowledge file: filename and the Gmail draft
  subject that carries it.
- For each fetch failure: URL and last error.

Keep the report terse — one line per item where possible.
