# granola-share-skill

A Claude Code skill that fetches your new Granola meeting transcripts, strips personal/sensitive content with the privacy filter, lets you review what was cut, and pushes the result to your private GitHub repo.

Designed for teams sharing filtered meeting context into a central repo. You stay in control: at every gate, the skill shows you what's about to happen and waits for you to approve before it does anything.

## What you need first

- **Claude Code** installed (`https://claude.com/claude-code`)
- **Granola MCP** wired up in your Claude Code setup, so the skill can fetch your meetings
- **GitHub CLI** (`gh`) installed and authed (`gh auth login`)
- **The [privacy-filter-skill](https://github.com/olhahah/privacy-filter-skill)** also installed — this skill calls it
- **A private GitHub repo** owned or shared with you, named `granola-<yourname>` (e.g., `granola-paul`)

## Install

Three commands, run once.

### 1. Clone the skill source repos

```bash
mkdir -p ~/skills
git clone https://github.com/olhahah/privacy-filter-skill ~/skills/privacy-filter
git clone https://github.com/olhahah/granola-share-skill ~/skills/granola-share
```

### 2. Symlink both into Claude Code's skill directory

```bash
mkdir -p ~/.claude/skills
ln -s ~/skills/privacy-filter/skill ~/.claude/skills/privacy-filter
ln -s ~/skills/granola-share/skill ~/.claude/skills/granola-share
```

The symlinks mean `git pull` in either source dir picks up new versions — no re-install step.

### 3. Clone your granola repo

```bash
git clone https://github.com/olhahah/granola-<yourname> ~/granola-<yourname>
```

(Olha will have invited you as a collaborator on this repo before you do this step.)

## Use

Once a week (or whenever you want), from inside your granola repo:

```bash
cd ~/granola-<yourname>
```

In Claude Code, invoke:

```
/granola-share
```

The skill walks you through it. Roughly:

1. Tells you how many transcripts you've already pushed and what's most recent.
2. Asks how far back to look (last 7 days / 30 / 90 / 6 months / year / since latest / all time / custom).
3. Lists the new meetings in that window — date, title, participants, duration.
4. Asks if you want to narrow the list (by keyword, project, etc.) and shows you what's in/out.
5. Asks if you want to drop any specific meetings — shows the final list.
6. Asks for final confirmation before any fetching happens.
7. Pulls each transcript, runs the privacy filter, writes one `.md` per meeting.
8. Stops and asks you to open `_logs/privacy-redactions.md` to review what was cut. You can revert anything important before the push lands. Loop until you're happy.
9. Commits and pushes.
10. Prints a summary and a link to the repo.

You can say `stop` at any gate.

## Update

Each skill updates independently. To pick up new versions:

```bash
cd ~/skills/granola-share && git pull
# or
cd ~/skills/privacy-filter && git pull
```

You don't have to update both at once.

## What gets pushed

Each meeting becomes one `.md` file in your repo's root, named `<YYYY-MM-DD>-<slug>.md`. The file has a YAML frontmatter with provenance metadata (Granola source ID, source URL, title, date, participants, meeting type) plus the filtered body.

The redaction log lives at `_logs/privacy-redactions.md`. It's also pushed — so Olha can review periodically what was cut.

## What does NOT get pushed

- Raw transcripts (those stay on your laptop only — filtered version is what reaches the repo)
- Meetings you didn't approve at the confirmation gate
- Redactions you chose to keep removed (the body has `[REDACTED]` where the personal content was)
- Anything outside the time window you chose

## Troubleshooting

| Problem | Fix |
|---|---|
| `gh auth status` says not logged in | `gh auth login` and follow the prompts |
| Skill says "not in a granola-* repo" | `cd` into your local clone of `granola-<yourname>` first |
| Skill says "privacy-filter skill not found" | You missed install step 2 for privacy-filter. Re-run the symlink commands. |
| Granola MCP tools unavailable | Your Claude Code isn't wired to Granola. See your Claude Code MCP settings. |
| Push fails with permission denied | You're trying to push to a repo you don't have write access on. Ask Olha to confirm your collaborator invite was accepted. |

## Related

- [privacy-filter-skill](https://github.com/olhahah/privacy-filter-skill) — the filter this skill calls

## Questions

Olha — olha.dolinska@healf.com
