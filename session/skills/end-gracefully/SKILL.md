---
name: end-gracefully
description: Check whether the current session can be closed cleanly — surfaces half-done work, unresolved questions, promises not kept, decisions made but never written down, unverified claims, temporary state left behind, and uncommitted changes, each with a recommended disposition. Read-only; reports and routes, never acts. Auto-invokes on "end gracefully", "can I close this", "anything I missed", "did we miss anything", "is this session done", "safe to close", "anything more then".
allowed-tools: [Read, Grep, Glob, Bash]
---

# End Gracefully — Pre-Close Sweep

Long sessions lose threads. This one looks back and says whether it's safe to
close — and for anything left, what to *do* about it.

**Read-only.** Never commits, never writes, never edits — it reports and routes.
That is what makes it safe to run at any moment.

**This runs when the user is done.** They are not looking for a report to read.
Be terse, decide for them, and let them confirm. A long output here is a failure.

## Step 1: Scan

Re-read the whole session, not just the recent part — buried items are the point.
Note the specific file, command, or question for each.

1. **Half-done** — started, completion never confirmed
2. **Unanswered** — a question raised, then overtaken and forgotten; includes a
   request the user made that was interrupted or superseded before it was done
3. **Promised** — something the assistant said it would do or add, and didn't
4. **Unrecorded** — a decision settled in conversation, nothing on disk
5. **Unverified** — asserted to work, never actually run; or a claim made to the
   user that later evidence contradicted and was never corrected
6. **Left behind** — temporary state still on the machine: background processes,
   tunnels, test jobs loaded into a scheduler, scratch repos or files, toggled
   configs, test data in a live system
7. **Deferred** — punted explicitly, captured nowhere

Then, if this is a git repo:

```bash
git status --short
git log --oneline @{upstream}..HEAD 2>/dev/null
```

If `docs/sessions/` exists, check for a log dated today — that answers "did we
already end this?" without the user looking. Optional signal only: never require
it, never create it.

## Step 2: Assign a disposition

Every item gets exactly one, chosen by you before showing it:

| Tag | Means |
|-----|-------|
| `DO` | An action that must happen before this counts as closed — undo temporary state, finish a promised step, fix something you found and told the user about. Name the exact action |
| `WRITE` | Must survive the session and is recorded nowhere a future session would look. An artifact can be the record — a dated, self-describing archive path needs no note; a rationale buried only in a commit message still does |
| `DROP` | Safe to neglect; say why, briefly |
| `?` | Genuinely the user's call — always attach your recommendation |

Default to `DROP` when unsure. `?` is for real forks, not for offloading the
thinking.

## Step 3: Report

**Clean — one line, stop:**

```
Safe to close. <what this session did, one clause>
```

**Not clean — verdict line, then one line per item. Max 5. No preamble:**

```
Not clean — 1 to do, 1 to write, 1 to decide, 1 droppable.

DO     Test job still loaded in the scheduler. → launchctl unload ~/Library/LaunchAgents/test.plist
WRITE  Decided to drop the retry layer (rate limit) — nothing on disk. → /handover:end
?      `webhook.ts` constructEvent stubbed. → recommend: note as next step, don't start now
DROP   3 modified files — all scratch output.

Say "do it" and I'll handle the DO and WRITE items — or name a ? and I'll do
what I recommended.
```

Rules for the output:

- Verdict line first, counts by tag
- One line per item: tag, what it is, then `→` the concrete action — a command,
  a file, a message to someone, a note. `/handover:end` is one possible route,
  not the default
- Hard cap of 5 items — if there are more, keep the 5 that matter and say
  "+N minor"
- No headings, no explanation of the categories, no closing summary
- Uncommitted work is reported, never committed

## Do not manufacture work

- Pre-existing backlog and standing machine state are not loose ends from this
  session — touching an area does not adopt its old risks, unless this session
  materially increased them
- A file read but not changed is not unfinished
- "Could be improved" is not outstanding
- Advisory sessions, and ones that answered their own question, are clean

**"Safe to close" is a correct answer, not a lazy one.** An item counts only if a
future session would be worse off not knowing it.
