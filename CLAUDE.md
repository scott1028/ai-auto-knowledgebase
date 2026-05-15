# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

Automated knowledge base managed by Claude. Per `README.md`, the repo stores knowledge entries created and maintained through Claude-assisted workflows — it is not an application codebase.

## Current state

The repo is a scaffold: only `README.md` exists. There is no build system, no test suite, and no source code. Do not invent commands or conventions beyond the rules below — when other questions arise (frontmatter, indexing, etc.), ask the user and update this file with the decisions once they exist.

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
- The YAML frontmatter `title:` field SHOULD be the Chinese title; you MAY append the original English title in parentheses if useful. `source_url`, `source_hash`, `retrieved` stay as-is. The filename slug stays in English (it derives from the source URL slug, not the title).
- Use 繁體中文 (zh-TW) conventions, not 簡體中文 (zh-CN) — e.g. 「資訊」not「信息」, 「程式」not「程序」, 「網路」not「网络」.
- Use full-width punctuation for Chinese text (，。：「」) and half-width punctuation inside code or English fragments. Do not mix.

This rule applies to NEW knowledge files going forward. Existing English files do not need to be retroactively translated unless the user asks.

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
