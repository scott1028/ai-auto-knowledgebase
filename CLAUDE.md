# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

Automated knowledge base managed by Claude. Per `README.md`, the repo stores knowledge entries created and maintained through Claude-assisted workflows — it is not an application codebase.

## Current state

The repo is a scaffold: only `README.md` exists. There is no build system, no test suite, and no source code. Do not invent commands or conventions beyond the rules below — when other questions arise (frontmatter, indexing, etc.), ask the user and update this file with the decisions once they exist.

## Scratch storage — `./.tmp/`

Any transient artifact you produce in this repo — `curl` / `wget` downloads, intermediate HTML, extracted JSON, scratch scripts, working buffers, anything that is NOT a final knowledge file — MUST be written under `./.tmp/`. Do NOT use `/tmp/`, the system temp dir, the repo root, or any path outside `./.tmp/`. The `.gitignore` already excludes `.tmp/*` (with `!.tmp/.keep`), so scratch never leaks into commits.

Default to `./.tmp/` silently — never ask the user where to cache a download or intermediate file. Examples:

```sh
curl -sSL "$URL" -o ./.tmp/page.html
wget -q "$URL" -O ./.tmp/page.html
```

Clean-up is optional (the directory is gitignored). If a run produces large files (>10 MB) you may delete them after the final commit succeeds. Never delete `./.tmp/.keep`, and never delete `./.tmp/pat.key` if it exists (it stores the GitHub PAT for the Contents-API fallback path).

## Knowledge entry rules

- All new knowledge markdown files MUST be placed in `.knowledge/`.
- Filenames MUST follow the format `YYYY-MM-DD_knowledge-title_<sourcehash>.md` (e.g. `2026-05-15_repo-bootstrap_a1b2c3d4.md`). The leading ISO date enables chronological sorting and time-based cross-referencing of knowledge.
- `<sourcehash>` is the first 8 hex characters of the SHA-256 of the source URL the knowledge was derived from. Generate it with: `printf '%s' "$URL" | sha256sum | cut -c1-8`. This deduplicates entries derived from the same source and lets you trace a file back to its origin.
- If the knowledge has no source URL (e.g. user-dictated, synthesized from conversation), omit the `_<sourcehash>` suffix entirely — do not fabricate a hash.
- Use the current date (not the date the underlying event occurred) as the filename prefix, unless the user specifies otherwise.
- Every knowledge file derived from a URL MUST embed that source URL inside the file itself, so readers can verify the original. Use YAML frontmatter at the top:

  ```markdown
  ---
  source_url: https://example.com/article
  source_hash: a1b2c3d4
  retrieved: 2026-05-15
  ---
  ```

  The `source_hash` value MUST match the `<sourcehash>` in the filename. For files with no source URL, omit the frontmatter (or set `source_url: null`).

### Language — Traditional Chinese (繁體中文)

All curated knowledge content MUST be written in Traditional Chinese (繁體中文 / zh-TW). The goal is reading speed: the user is a native zh-TW reader and skims faster in Chinese than in long-form English. This rule applies to the synthesized prose you write — section headings, summaries, explanations, bullet points, takeaways — not to source-faithful fragments.

Specifically:

- Translate or paraphrase the source into 繁體中文 rather than mirroring the original English structure. A short Chinese summary that captures the point beats a verbatim English translation.
- DO NOT translate: identifiers that must stay verbatim — code blocks, CLI commands, function/class/API names, URLs, file paths, error messages, RFC/spec terms, brand/product names (e.g. `View Transitions`, `Intl.Segmenter`, `Temporal`, `MDN`). Keep these in their original English form inline.
- Technical terms with a well-established Chinese rendering should use that rendering, with the English term in parentheses on first mention if it aids lookup — e.g. 「視圖轉場 (View Transitions)」, 「無障礙 (accessibility)」. After first mention, prefer the Chinese form.
- The YAML frontmatter `title:` field SHOULD be the Chinese title; you MAY append the original English title in parentheses if useful. `source_url`, `source_hash`, `retrieved` stay as-is.
- **Filename slug stays in English.** The `knowledge-title` segment of `YYYY-MM-DD_knowledge-title_<hash>.md` is derived from the source URL's slug (or an English keyword summary for non-URL knowledge), NOT translated. This keeps filenames shell-safe, sort-stable, and trivially greppable. Only the *content* and the frontmatter `title:` are in Chinese.
- Use 繁體中文 (zh-TW) conventions, not 簡體中文 (zh-CN) — e.g. 「資訊」not「信息」, 「程式」not「程序」, 「網路」not「网络」.
- Use full-width punctuation for Chinese text (，。：「」) and half-width punctuation inside code or English fragments. Do not mix.

This rule applies to NEW knowledge files going forward. Existing English files do not need to be retroactively translated unless the user asks.

### Diagrams — ASCII only

When a diagram helps human comprehension (flowcharts, architecture diagrams, sequence diagrams, state machines, layouts, hierarchies, etc.), draw it as **ASCII art inside a fenced code block**. Output files are plain Markdown (`.md`) consumed by humans reading the file directly — Mermaid, PlantUML, or other diagram DSLs do NOT render in this context and just appear as confusing source code. ASCII renders predictably everywhere (GitHub, VS Code preview, terminal, plain text).

Rules:

- Wrap the diagram in a fenced code block — use a plain ```` ``` ```` fence (no language tag) or ```` ```text ```` so monospace alignment is preserved.
- Use a monospace-friendly box-drawing vocabulary. Pick ONE style per diagram and stay consistent:
  - ASCII only: `+`, `-`, `|`, `>`, `<`, `^`, `v` for boxes and arrows. Maximum portability.
  - Unicode box-drawing (`─ │ ┌ ┐ └ ┘ ├ ┤ ┬ ┴ ┼ → ← ↑ ↓`) is also OK and often clearer. Don't mix the two styles within one diagram.
- Keep the diagram narrow enough to read without horizontal scroll — aim for ≤80 columns. If it grows beyond that, split it into multiple smaller diagrams or simplify.
- Label every box and arrow. An unlabelled diagram is rarely worth the screen space.
- Prefer prose for anything a 3-box diagram can't convey — diagrams should clarify, not pad. If you can say it in one sentence, do that instead.
- Do NOT use Mermaid (```` ```mermaid ````), PlantUML, Graphviz, or other rendered-diagram DSLs. They don't render in raw Markdown viewers and degrade to unreadable source.

Example:

```
+----------+    fetch    +-----------+    parse    +-----------+
|  source  | ----------> |  fetcher  | ----------> |  encoder  |
|   URL    |             |  (curl)   |             |  (codec)  |
+----------+             +-----------+             +-----------+
                                                         |
                                                         v
                                                   +-----------+
                                                   | .knowledge|
                                                   +-----------+
```

### Source images — convert to ASCII when possible, otherwise summarise

Knowledge files are plain Markdown read directly from the filesystem; embedded raster images (PNG, JPG, GIF, WebP, AVIF, SVG) are unreliable for the same reason rendered-diagram DSLs are: the reader may be viewing in a terminal, raw text editor, or pager where images don't render at all. Worse, blindly carrying over `![alt](src)` references creates broken links because the original `src` is a relative path on the source site, not in this repo.

Handling priority when the source contains an image:

1. **If the image conveys structural information** (architecture diagrams, flowcharts, state machines, UI layouts, axis-labelled graphs, sequence diagrams, tables of relationships, code-comparison screenshots): **reconstruct it as an ASCII diagram** per the rules above. The structural information is what the reader actually needs; the pixel rendering is incidental.
2. **If the image is illustrative but non-structural** (photos, decorative banners, author avatars, generic stock graphics, hero images): **drop it entirely.** Do not include a placeholder, do not include the original URL, do not include the alt text alone. These add noise without value.
3. **If the image is data the reader genuinely needs to see** (a specific chart you cannot reasonably ASCII-fy, a screenshot of a unique UI bug, etc.): **summarise its content in prose** describing what it shows, then link out to the source URL once at the top of the file (which the `source_url` frontmatter already provides). Do NOT embed `![](https://...)` pointing at the source host — those links rot, and external image hotlinking is fragile.
4. **NEVER copy an `<img>` tag or `![](...)` markdown directly from the source.** Both forms produce broken or non-rendering output in plain Markdown viewers.
5. **Do NOT download images into `.knowledge/`.** Binary blobs don't belong in the knowledge directory. If a reader really needs the image, they can follow `source_url`.

In practice, most source images in technical articles fall into category 1 or 2. Category 3 should be rare. When in doubt, ASCII-fy and prose-summarise; the goal is a self-contained text file that reads correctly anywhere.

## Ingestion workflow (dedup before fetch)

Before fetching, processing, or writing knowledge from a source URL:

1. Compute `<sourcehash>` from the URL: `printf '%s' "$URL" | sha256sum | cut -c1-8`.
2. Check whether any existing file in `.knowledge/` already ends with `_<sourcehash>.md`: `ls .knowledge/*_<sourcehash>.md 2>/dev/null` (or use Glob with pattern `.knowledge/*_<sourcehash>.md`).
3. If a match exists, SKIP the URL entirely — do not re-fetch, do not overwrite, do not create a new dated copy. Record the skip (URL + existing filename) in memory.
4. If no match exists, proceed with fetching and writing the new file.

When processing multiple URLs in one turn, perform the dedup check for every URL up front, then at the end of the response include a short message listing every URL that was skipped and which existing file it matched. Do not omit this notice — the user relies on it to know what was reused vs newly fetched.

## Recent updates index (`./recent_updates.md`)

`./recent_updates.md` (at repo root, NOT inside `.knowledge/`) is a fast-lookup index of every knowledge file added in the last 30 days. It exists so dedup checks can scan one small file instead of the whole `.knowledge/` directory.

Format — one filename per line, newest first, no other content:

```
2026-05-15_repo-bootstrap_a1b2c3d4.md
2026-05-14_some-article_ef567890.md
...
```

Maintenance rules:

1. **Dedup fast path:** before the filesystem check in step 2 of the ingestion workflow, grep `recent_updates.md` for `_<sourcehash>.md`. A hit there is authoritative — skip immediately. A miss does NOT prove absence (the file may be older than 30 days), so still fall back to the `.knowledge/*_<sourcehash>.md` check.
2. **After writing a new knowledge file:** prepend its filename to `recent_updates.md`.
3. **Pruning (FIFO):** when updating the index, walk entries from the bottom (oldest) upward and drop them one-by-one as long as their `YYYY-MM-DD` prefix is older than 30 days from today. Stop at the first entry within the 30-day window — because the list is kept newest-first, everything above is guaranteed to be in-window. Compare dates lexicographically; the ISO format makes this safe.
4. Do not edit `recent_updates.md` for any other reason. Do not add comments, headers, or blank lines.

## README "Last Updated" stamp

Whenever the knowledge base is mutated (new file written to `.knowledge/`, existing entry edited, or `recent_updates.md` updated), also update the date under the `## Last Updated` heading in `README.md` to today in `YYYY-MM-DD` format. Replace the existing date line in place — do not append a new line, do not add a time component, do not add prose. If multiple knowledge changes happen in one turn, update the stamp once at the end. Skip the stamp update if nothing in `.knowledge/` or `recent_updates.md` actually changed.

## Commit safety — secret scan

Before any `git commit`, `git push`, or GitHub Contents-API write that touches this repo, run a secret-pattern scan on the change set. This applies regardless of what the commit is for (knowledge ingest, doc edit, config tweak — anything).

Required checks before any push or API write:

1. **`.gitignore` covers `./.tmp/`**: verify `.tmp/*` (or `./.tmp/pat.key` specifically) is gitignored. If not, add it and commit that change FIRST.
2. **Scan working tree (outside `./.tmp/`)** for PAT-shaped strings:

   ```sh
   grep -rIE 'gh[pousr]_[A-Za-z0-9_]{20,}|github_pat_[A-Za-z0-9_]{20,}' \
        --exclude-dir=.git --exclude-dir=.tmp .
   ```

3. **Scan staged diff** for the same patterns:

   ```sh
   git diff --cached | grep -E 'gh[pousr]_[A-Za-z0-9_]{20,}|github_pat_[A-Za-z0-9_]{20,}'
   ```

If either scan matches anything in the staging area or working tree outside `./.tmp/`:

- **ABORT immediately.** Do not push, do not retry, do not "clean it up after the push."
- Surface a loud warning to the user verbatim: `⚠️ GITHUB PAT DETECTED IN REPO — push aborted`.
- Do not proceed until the leak is removed from the working tree AND, if the bad value was already committed locally, the affected commits are rewritten (e.g. `git reset` + re-stage cleanly).

Tokens stored in `./.tmp/pat.key` are fine — that path is gitignored. The scan only flags PATs that have escaped that directory.

## Push protocol — SSH probe

When probing whether SSH-based git operations work in this environment, use a **real behaviour check**, not `ssh -T`:

```sh
git rev-parse --is-inside-work-tree                    # must print "true"
git remote get-url origin | grep -E '^git@github\.com' # must match (SSH form)
git ls-remote origin HEAD                              # exit 0 = SSH + transport OK
```

`git ls-remote origin HEAD` is the canonical probe — it authenticates and reaches the remote, so a success here proves `git push` will work. Do NOT use `ssh -T git@github.com`: it has been observed to fail (`Permission denied (publickey)`) on environments — notably WSL — where the SSH agent forwards keys to real git operations but not to the bare `-T` test connection. A failed `ssh -T` is therefore NOT evidence that SSH is unusable.

You may see harmless `kwalletd` / DBus error lines interleaved with successful git output on Linux/WSL, e.g. `Couldn't start kwalletd: QDBusError(...)`. These are credential-helper noise and do NOT indicate failure — only the git command's exit code and final output (`HEAD` ref hash for `ls-remote`, refspec range for `push`) determine success. Do not interpret these as auth errors.
