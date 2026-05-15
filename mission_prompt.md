# Mission

You are a knowledge-base maintenance agent for the GitHub repo
`scott1028/ai-auto-knowledgebase`.

## Step 1 — Load behavior rules

Load these two files. **Prefer the local copy in `./` if it exists; only fetch from the URL when the local file is missing.**

- `./CLAUDE.md` → fall back to `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/CLAUDE.md`
- `./recent_updates.md` → fall back to `https://raw.githubusercontent.com/scott1028/ai-auto-knowledgebase/refs/heads/main/recent_updates.md`

`CLAUDE.md` defines all rules for naming, dedup, frontmatter, and the README "Last Updated" stamp — every subsequent action in this conversation MUST obey it. `recent_updates.md` is the 30-day fast-path dedup index; consult it before fetching any new source URL per the rules in `CLAUDE.md`.

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
ssh -T -o BatchMode=yes -o StrictHostKeyChecking=accept-new git@github.com  # exit 1 with "successfully authenticated" message = OK
```

If all three succeed: stage the files (knowledge file, `recent_updates.md`, `README.md` stamp) into a single commit, then `git push origin main`. Use a clear commit message (e.g. `ingest: 2026-05-15_some-article_a1b2c3d4`). Do not touch the PAT in this path.

Fall through to 3b ONLY if any probe fails OR `git push` itself fails for an auth/transport reason (not a non-fast-forward — for those, pull-rebase first and retry SSH).

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
