# session

Skills that operate on the **conversation** rather than on project files. That's
the line between this plugin and [`handover`](../handover/): handover reads and
writes `WORK.md`, `TODO.md`, and session logs; these skills look at what actually
happened in the session and report on it.

```bash
/plugin install session@pongsapakl-skills
```

## Skills

| Command | What it does |
|---------|--------------|
| `/session:end-gracefully` | Checks whether the session can be closed cleanly — half-done work, unresolved questions, promises not kept, unrecorded decisions, unverified claims, temporary state left behind, uncommitted changes. Each item comes with a recommended disposition. Read-only. |

## Why it isn't part of handover

It needs to work in sessions that never ran `/handover:init` — a one-off in a
scratch directory, or a repo with no docs structure. It uses `WORK.md` and
`docs/sessions/` as extra signal when they exist, but never requires or creates
them.

Session logs showed the reverse case too: roughly half the "did I miss
something?" asks came from projects that *did* have handover fully set up.
Having the files didn't prevent losing track of a thread, so the check is
independent of the file contract.

## Design

**Read-only, always.** It never commits, never writes docs, never edits. It
reports and points at whatever handles each item — `/handover:end` for logging,
a commit for uncommitted work. That's what makes it safe to run mid-session.

**It runs when you're done, so it has to be short.** Verdict line, then one line
per item, capped at five. Every item carries a disposition decided in advance —
`DO` (an action before closing), `WRITE` (must survive), `DROP` (safe to
neglect), or `?` (your call, with a recommendation attached) — so closing out is confirming, not triaging.

**It must be willing to say "safe to close."** The obvious failure mode is
inventing loose ends to look thorough. Pre-existing backlog isn't a loose end;
"could be improved" isn't outstanding. An item only counts if a future session
would be worse off not knowing it.
