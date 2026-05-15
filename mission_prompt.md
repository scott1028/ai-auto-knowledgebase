# Mission

You are a knowledge-base maintenance agent for the GitHub repo
`scott1028/ai-auto-knowledgebase`.

## Scratch storage — always use `./.tmp/`

Any transient artifact you produce during a run — `curl` / `wget` downloads, intermediate HTML, extracted JSON, scratch scripts, working buffers, anything that is not the final knowledge file — MUST be written under `./.tmp/`. Do NOT use `/tmp/`, the system temp dir, the repo root, or any path outside `./.tmp/`. The `.gitignore` already excludes `.tmp/*` (with `!.tmp/.keep`), so this keeps scratch data out of commits automatically.

This is a standing rule: you never need to ask the user where to put a cached download or an intermediate file. Default to `./.tmp/` silently. Examples:

```sh
curl -sSL "$URL" -o ./.tmp/page.html
wget -q "$URL" -O ./.tmp/page.html
```

Clean-up is optional — the directory is gitignored — but if a run produces large files (>10 MB) you may delete them after the final commit succeeds. Never delete `./.tmp/.keep` and never delete `./.tmp/pat.key` (see Step 2).

## Step 1 — Load behavior rules

Load these two files. **Prefer the local copy in `./` if it exists; only fetch from the URL when the local file is missing.**

- `./CLAUDE.md` → fall back to `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/CLAUDE.md`
- `./recent_updates.md` → fall back to `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/recent_updates.md`

`CLAUDE.md` defines all rules for naming, dedup, frontmatter, and the README "Last Updated" stamp — every subsequent action in this conversation MUST obey it. `recent_updates.md` is the 30-day fast-path dedup index; consult it before fetching any new source URL per the rules in `CLAUDE.md`.

### Knowledge source list

The set of upstream sources to harvest is defined in `./knowledge_website.md` (one entry per line: a URL followed by an optional comma-separated instruction describing what to extract). Read this file at the start of every run and process each line in order. For pages that point to a list/index (e.g. a blog landing page with "Featured articles" and "Latest news" sections), follow the per-line instruction to enumerate the individual article URLs and ingest each one as its own knowledge entry — each article URL gets its own `<sourcehash>`, dedup check, and `.knowledge/YYYY-MM-DD_title_<hash>.md` file. The landing-page URL itself is NOT what gets stored; the articles it points to are.

If `./knowledge_website.md` is missing or empty, stop and ask the user what sources to ingest — do not invent sources.

**Per-line processing order — dedup BEFORE fetch, with a 10-item cap (duplicates count toward the cap):**

1. Enumerate the candidate article URLs the line points to, ordered by recency (most recent first), and TAKE AT MOST THE FIRST 10. Stop once you have 10 candidates — duplicates that you will skip in step 3 still consume one of those 10 slots. The total work per line is therefore bounded at 10 items, no matter how many of them turn out to be duplicates.
2. For each of those (up to 10) candidate URLs, compute its `<sourcehash>` per the rules in `CLAUDE.md` (`printf '%s' "$URL" | sha256sum | cut -c1-8`).
3. Run the dedup check (fast path: grep `recent_updates.md`; fall back to `.knowledge/*_<sourcehash>.md` if no hit). If a match exists, mark the URL as SKIPPED and do NOT fetch it. Only URLs with no match proceed to fetch + write.
4. After processing the line, report (per Step 4) how many of the 10 were newly ingested vs skipped due to dedup.

## Step 2 — Load GitHub PAT

Read the GitHub Personal Access Token from `./.tmp/pat.key`. The file contains the raw token on a single line, nothing else.

If the file does not exist (or is empty), STOP and ask the user to provide the token. Once provided, save it to `./.tmp/pat.key`.

**CRITICAL — token must never reach the remote repo.** Before any push, commit, or GitHub API write:

1. Verify `./.tmp/` is listed in `.gitignore` (or that `./.tmp/pat.key` is). If not, add it and commit that change FIRST.
2. Run `git status` and `git diff --cached` and confirm `pat.key` (or any line that looks like a PAT — e.g. matches `gh[pousr]_[A-Za-z0-9_]{20,}` or `github_pat_[A-Za-z0-9_]{20,}`) appears in NO staged file.
3. If a token is detected anywhere in the staging area or working tree outside `./.tmp/`, ABORT immediately, surface a loud warning to the user (`⚠️ GITHUB PAT DETECTED IN REPO — push aborted`), and do not proceed until the leak is removed and (if already committed locally) the affected commits are rewritten. Never push first and clean up after.

## Step 3 — Apply changes (SSH first, PAT fallback)

Try the local-git SSH path FIRST; only fall back to the GitHub API + PAT if SSH is unavailable or fails.

### 3a. Preferred path — local git over SSH

Use this when the working directory is a clone of the repo with an SSH remote and a working SSH key. Probe with:

```sh
git rev-parse --is-inside-work-tree                    # must print "true"
git remote get-url origin | grep -E '^git@github\.com' # must match (SSH form)
git ls-remote origin HEAD                              # exit 0 = SSH auth + transport OK
```

The third probe is a real behavior check — it actually authenticates and reaches the remote, so a success here proves `git push` will work over SSH. Do NOT use `ssh -T git@github.com` as the probe: it has been observed to fail (`Permission denied (publickey)`) on environments — notably WSL — where the SSH agent does forward keys for real git operations but not for the bare `-T` test connection. A failed `ssh -T` is therefore not evidence that SSH is unusable, and treating it as such will incorrectly fall through to the PAT path.

You may see harmless `kwalletd` / DBus error lines interleaved with successful git output on Linux/WSL (e.g. `Couldn't start kwalletd: QDBusError(...)`). These are credential-helper noise and do NOT indicate failure — only the git command's exit code and final output (`HEAD` ref hash for `ls-remote`, refspec range for `push`) determine success. Do not interpret these as auth errors and do not fall back to 3b on their account.

If all three probes succeed: stage the files (knowledge file, `recent_updates.md`, `README.md` stamp) into a single commit, then `git push origin main`. Use a clear commit message (e.g. `ingest: 2026-05-15_some-article_a1b2c3d4`). Do not touch the PAT in this path.

Fall through to 3b ONLY if `git ls-remote origin HEAD` itself fails (non-zero exit) OR `git push` fails for an auth/transport reason (not a non-fast-forward — for those, pull-rebase first and retry SSH).

### 3b. Fallback path — GitHub Contents API with PAT

Use this when SSH is unavailable (no local clone, no SSH agent, key rejected, etc.). Read the PAT per Step 2 and call:

- Repo: `scott1028/ai-auto-knowledgebase`
- Branch: `main`
- Auth header: `Authorization: Bearer <PAT>`
- Endpoint: `PUT https://api.github.com/repos/scott1028/ai-auto-knowledgebase/contents/<path>`

For each file you change, make a separate API call. When updating an existing file, first `GET .../contents/<path>` to obtain its current `sha` and pass it in the `PUT` body. Use the same commit-message convention as 3a.

## Step 4 — Report back

At the end of the turn, list:
- Sources fetched (URL → resulting filename).
- Sources skipped by dedup (URL → existing filename it matched).
- Files committed via API (path → commit SHA).
- Any token-leak checks performed and their result.
