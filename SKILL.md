---
name: granola-share
description: Use this skill to fetch the user's new Granola meeting transcripts, strip personal and sensitive content via the privacy filter, let the user review what was cut, and push the filtered .md files to their private granola-* GitHub repo. The skill is interactive at every gate — it shows the full list of what's being included/excluded and waits for the user to approve before pushing. Triggers on requests like "push my new meetings", "share my granola transcripts", "sync my meetings to the repo", "/granola-share".
---

# Granola share — fetch, filter, review, push

## What this skill does

Fetches the user's new Granola meeting transcripts, runs them through the privacy filter, lets the user review what was cut, and pushes filtered `.md` files to their private `granola-<name>` GitHub repo.

The skill is **interactive at every consequential gate**. At each step, it shows the user the full list of what's being included or excluded — never silently defaults. The user must approve before anything is fetched, before anything is pushed, and before any redaction is locked in.

## When to use

When the user asks to:
- push their new Granola meetings to GitHub
- share their meeting transcripts to the team repo
- sync their Granola meetings
- run `/granola-share`

## What you need before starting

Check these before doing anything else. If any are missing, stop and tell the user what to set up.

1. **Working directory is a clone of a `granola-*` GitHub repo.**
   - Run `git rev-parse --show-toplevel` to confirm we're in a git repo
   - Run `git remote get-url origin` — the origin URL must end with `granola-<something>.git` or `granola-<something>` (with or without `.git`)
   - If not, tell the user: "Run this from inside your local clone of your granola-<your-name> repo. If you haven't cloned it yet: `git clone https://github.com/olhahah/granola-<yourname> ~/granola-<yourname>` and try again from inside that directory."

2. **`gh` CLI is authed.**
   - Run `gh auth status`. If not authed, tell the user to run `gh auth login` and try again.

3. **Granola MCP tools are reachable.**
   - The skill uses `mcp__claude_ai_Granola__list_meetings` (or `get_meetings`) and `mcp__claude_ai_Granola__get_meeting_transcript`. If these tools aren't available in the current Claude Code session, tell the user: "I can't see your Granola — make sure the Granola MCP is wired up in your Claude Code setup."

4. **`privacy-filter` skill is installed.**
   - Check that `~/.claude/skills/privacy-filter/SKILL.md` exists.
   - If missing, stop and tell the user: "I need the privacy-filter skill to be installed before pushing. Install it: `mkdir -p ~/.claude/skills && git clone https://github.com/olhahah/privacy-filter-skill ~/.claude/skills/privacy-filter`. Then re-run me."
   - **Never push unfiltered transcripts.** If the filter isn't installed, the skill stops — no silent fallback.

## How it works — the gates

The skill walks the user through these gates in order. At every gate, the user can say `stop` to bail. The skill does nothing irreversible until the final push step.

### Gate 1: Take stock of what's already in the repo

For every `.md` file in the repo root (skip `_logs/`, `README.md`, hidden files):
- Parse the YAML frontmatter
- Collect the `source_id` value into a set (the "seen-set")
- Note the most recent `date:` value

Print to the user:

> "Repo currently has **{N} transcripts** pushed, most recent dated **{YYYY-MM-DD}**." (If N is 0, say "Repo is empty.")

### Gate 2: Ask how far back to look

Ask the user to pick a time window. **Always offer the same full menu** — never anchor the suggestion on what's in the repo.

```
How far back should I look for new Granola meetings?
  1) Since the latest meeting already in the repo (incremental top-up)
  2) Last 7 days
  3) Last 30 days
  4) Last 90 days
  5) Last 6 months
  6) Last year
  7) All time
  8) Custom date — type a date like "since 2025-09-01"

(Pick a number or describe in your own words.)
```

If the repo is empty, omit option 1.

**Critical rule:** option 1 is never a silent default. If a user has 11 months of un-uploaded meetings and their newest existing transcript is from last week, "since latest" would silently skip all of them. The user must choose option 1 explicitly.

Parse the user's response and compute `since_date`.

### Gate 3: Fetch the candidate meeting list

Call `mcp__claude_ai_Granola__list_meetings` (or `get_meetings`) for the chosen window. For each meeting, capture: id, title, date, participants, duration, any folder/tag info.

### Gate 4: Dedupe and report

Filter out meetings whose id is in the seen-set.

Print:

> "Found **{found}** meetings in window. **{already}** are already in the repo. **{new}** are new."

If `new` is 0, say "Nothing new to push. Done." and stop.

### Gate 5: Show the new-meetings list

Print a numbered list of all `new` meetings. Format:

```
NEW MEETINGS:

  1. 2026-05-21 — Paul / Marion (45 min) — participants: Paul, Marion — [folder: roadmap]
  2. 2026-05-22 — Pod review (60 min) — participants: Hamza, Paul, Marion, Olha
  3. 2026-05-22 — Random 1:1 (15 min) — participants: Paul, Sasha
  ...
```

No filters applied yet. The user sees everything that's in the window.

### Gate 6: Optional inclusion filter

Ask:

> "Want me to narrow this list? E.g., only meetings whose title contains a keyword, or only meetings in a certain Granola folder. Type `all` to keep everything, or describe the filter (e.g., 'only meetings about roadmap', 'only 1:1s with Paul')."

If the user describes a filter:
- Apply it (you may need to interpret loosely — "only 1:1s" means meetings with 2 participants; "only roadmap meetings" means title or folder contains "roadmap"; etc.)
- Show **both lists** clearly labelled — INCLUDED and EXCLUDED:

```
INCLUDED (3):
  1. 2026-05-21 — Paul / Marion (45 min)
  2. 2026-05-25 — Paul / Olha (30 min)
  3. 2026-05-27 — Paul / Hamza (45 min)

EXCLUDED (4):
  - 2026-05-22 — Pod review (60 min) — not a 1:1
  - 2026-05-22 — Roadmap workshop (90 min) — not a 1:1
  ...

Does this look right? (`yes` / describe a different filter / `clear filter` to start over)
```

Loop until the user says `yes` or `all` or `clear filter`.

### Gate 7: Optional exclusion of specific meetings

Show the (possibly filtered) list with index numbers and ask:

> "Want to drop any specific meetings before we proceed? E.g., `drop 3, 7` to remove specific ones, or `none` to keep all. (You'll get another chance to bail before push.)"

If the user says `drop X, Y`, remove those indices and re-show the list:

```
FINAL LIST (2):
  1. 2026-05-21 — Paul / Marion (45 min)
  2. 2026-05-27 — Paul / Hamza (45 min)
```

Loop until the user confirms.

### Gate 8: Confirm before any Granola fetch

Print:

> "About to fetch {N} transcripts from Granola and run them through the privacy filter. Nothing will be pushed yet — you'll review the redactions before push. Proceed? (`yes` / `stop`)"

If `stop`, exit. If `yes`, proceed.

### Gate 9: Fetch and filter each meeting

**First, a one-time safety check.** Make sure the repo's `.gitignore` lists `_logs/` and `.granola-share/` (create the file or append the lines if missing). This guarantees that even if some older tool writes a log into the repo, it can never be committed. The redaction log itself lives outside the repo (see step 2) — this is belt-and-braces.

For each meeting in the final list, in order:

1. **Fetch the full transcript** via `mcp__claude_ai_Granola__get_meeting_transcript`.

2. **Apply the privacy filter.** Read the rules from `~/.claude/skills/privacy-filter/SKILL.md` and apply them to the transcript text. **Write the redaction log to a local-only path *outside* this repo**, so the removed content never reaches GitHub:

   ```
   ~/.granola-share/<repo-name>/privacy-redactions.md
   ```

   `<repo-name>` is this repo's folder name — the basename of `git remote get-url origin`, minus any trailing `.git` (e.g. `granola-taylor`). Create the folder first with `mkdir -p`. This log holds the *full quotes* of everything stripped out — third-party medical info, mental-health detail, personal anecdotes. It must never be written inside the repo and never committed. The contributor reviews it on their own machine (Gate 10); it stays with them.

3. **Build the file:**
   - **Filename:** `<YYYY-MM-DD>-<slug>.md`. Slug = the title lowercased, with non-alphanumeric characters collapsed to `-`, then trimmed of leading/trailing `-`, max 50 chars.
   - **YAML frontmatter:**
     ```yaml
     ---
     source_type: granola_transcript
     source_id: <Granola meeting UUID>
     source_url: <Granola URL, typically https://notes.granola.ai/t/<id>>
     title: <meeting title as Granola has it>
     date: <ISO 8601 with timezone, from Granola>
     participants:
       - <name>
       - <name>
     meeting_type: <see below>
     fetched_at: <today's date in YYYY-MM-DD>
     ---
     ```
   - **`meeting_type` inference:**
     - If participants count == 2 → `1:1`
     - Otherwise → `unknown` (Olha can refine on her side)
   - **Body:** `# Filtered transcript — <title>, <pretty date>\n\n<filtered text from the privacy filter>`

4. Write the file to the repo root (do not commit yet).

5. Print a per-meeting progress line: "Filtered {i}/{N}: {filename} ({redactions in this meeting} redactions)".

### Gate 10: Final review of the redaction log

Once all transcripts are filtered, **stop and let the user check what was cut**.

Print:

> "All {N} transcripts are filtered and written locally. **{total redactions} redactions** total ({high-confidence count} high confidence, {low-confidence count} low confidence) across the {M} meetings that had any.
>
> Before I push, please open your local redaction log — `~/.granola-share/<repo-name>/privacy-redactions.md` (it's outside the repo and stays on your computer; it's never pushed) — and skim through. Anything important the filter cut that you want to keep? Tell me which to restore (e.g., 'restore redaction 3 in the 14 May meeting', 'keep the bit about the customer escalation', 'revert all low-confidence redactions in meeting 2'), or say `push` if it looks good."

Wait for the user's response. Possible responses:

- **`push`** — proceed to commit and push.
- **`stop`** — abort. The local files stay (so the user can manually inspect or finish later); nothing is committed.
- **Restore instructions** — for each restoration:
  - Find the matching log entry (by meeting + quote text + index)
  - Re-insert the original removed text into the filtered transcript at the right position (replace the `[REDACTED]` marker for partial redactions, or re-insert the full segment where it originally lived for whole-segment cuts)
  - Remove the corresponding entry from the log file
  - Re-count redactions and print the updated total
  - Show the updated log location: "Updated. {new total} redactions remaining. Anything else, or `push`?"

Loop until the user says `push` or `stop`.

### Gate 11: Commit and push

When the user says `push`:

1. Determine git author identity (do **not** modify global git config):
   - If `git config user.name` and `git config user.email` are set, use them.
   - Otherwise, derive from `gh api user`: name = `.name // .login`, email = `<id>+<login>@users.noreply.github.com`.

2. Stage and commit — stage **only** the transcript files written this run, never a blanket `git add .`:
   ```bash
   git add -- <file1>.md <file2>.md …   # the exact filenames written in Gate 9
   git -c user.name="<name>" -c user.email="<email>" commit -m "Add <N> transcript(s) — <range of dates>"
   ```
   The redaction log lives outside the repo, so it can't be staged here anyway — but never use `git add .`, which could sweep in stray local files.

3. Push:
   ```bash
   git push origin main
   ```

4. If push fails (auth, network, etc.), print the error verbatim and stop — do not retry silently.

### Gate 12: Report

Print:

```
Done.

Pushed {N} transcripts to https://github.com/{owner}/{repo}:
  - 2026-05-21-paul-marion.md
  - 2026-05-27-paul-hamza.md

{final redactions count} redactions made after your review.

To check what landed: `gh repo view {owner}/{repo} --web`
```

## Safety rules — non-negotiable

1. **Never push without explicit user confirmation.** Always show the list and ask at every gate.
2. **Never push if privacy-filter isn't installed.** Hard stop. No fallback to "raw push".
3. **Never push if the filter errors on a meeting.** If the filter raises an exception or produces unparseable output, stop after that meeting and surface the error.
4. **Skip meetings the user declined.** Once they say "drop 3" at any gate, do not push #3.
5. **Idempotent on dedupe.** Re-running on the same set of meetings (matching `source_id`s already in the repo) is a no-op — Gate 4 dedupes them.
6. **Don't auto-resolve auth issues.** If `gh` isn't authed or git push fails, tell the user — don't try to fix it silently.
7. **Don't modify global git config.** Use `git -c user.name=... -c user.email=...` per commit.
8. **Never write or commit the redaction log inside the repo.** It holds the full removed quotes and stays local on the contributor's machine (`~/.granola-share/<repo-name>/privacy-redactions.md`). Stage only the filtered transcript files — and the filtered files keep `[REDACTED]` markers, never the original text.

## Plain language for outputs

- Don't say "deduplicating against the seen-set" — say "skipping meetings that are already in the repo".
- Don't say "applying redaction policy" — say "running the privacy filter".
- Don't say "provenance frontmatter" — say "the metadata header (Granola source, date, participants)".
- Don't say "idempotent" — just don't bring it up; it's not user-relevant.

## Edge cases

- **The user picks "all time" and there are thousands of meetings.** Print the count before fetching transcripts and re-confirm. "Found 2,400 meetings in window. {already} already in repo, {new} new. That's a lot — really pull transcripts for all {new}? (`yes` / pick a smaller window)."
- **A meeting in Granola has no transcript yet (still processing).** Skip it with a one-line note: "Skipping {title} — Granola hasn't finished transcribing." Don't fail the whole run.
- **A meeting's date is in the future** (calendar-only, no transcript). Skip with a note.
- **The user describes an inclusion filter you can't easily interpret.** Ask for clarification — show one or two interpretations and let them pick. Better to clarify than to apply the wrong filter silently.
- **A reversion target is ambiguous** (e.g., "restore the bit about the customer" matches three redactions). Ask the user to be more specific or show numbered candidates.
- **Repo is on a branch other than `main`.** Push to whatever branch is currently checked out. Don't switch branches.
- **There are uncommitted changes already in the repo when the skill starts.** Tell the user: "You have uncommitted changes in this repo: [list]. Should I include them in this push, or stash them first? (`include` / `stash` / `stop`)".

## Related files

- **The privacy filter skill** (read at runtime): `~/.claude/skills/privacy-filter/SKILL.md`
- **The redactions log** (created/appended at runtime, local-only — never pushed): `~/.granola-share/<repo-name>/privacy-redactions.md`
