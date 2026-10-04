---
name: swiss-cheese
description: Audit a piece of work (a design, an implementation, a plan, a config, an infra or backup setup) with two or more independent fresh-context reviewers on different models, then merge their findings into one ranked list the user can act on. The Swiss-cheese mentality: we don't know what we don't know, so stack independent layers that each catch different holes. Like a code review, but for broader work. Read-only; reports and recommends, never edits. Auto-invokes on "swiss cheese", "audit this", "fresh eyes on this", "second opinion on this design/setup", "what am I missing", "we don't know what we don't know", "verify everything is ok", "consult another model".
allowed-tools: [Agent, Read, Grep, Glob, Bash]
---

# Swiss Cheese — independent audit of a piece of work

One reviewer sees one set of holes. Two reviewers with different training, no shared context, and the same brief see different ones; what both miss is rarer. This skill runs that layering on whatever the session just built or decided, and hands back one ranked list.

**Read-only, always.** The auditors never edit, never run anything that changes state, never read secrets. This skill reports; the user decides what to fix.

## Step 1: Scope the audit (one sentence back to the user, no question unless needed)

Write down, for yourself, exactly what is under review and where it lives:
- the artefacts: files, directories, docs, scripts, configs, decision records; name the paths in the order an auditor should read them (overview first, then decisions, then implementation, then logs or evidence of what actually ran);
- the claim being made ("this backup design keeps three copies and the client can't delete them", "this migration script is idempotent", "this plan covers rollback");
- what is out of scope.

If the user named the target, don't ask; if the session has several candidates, pick the most recent substantial piece of work and say so in one line.

## Step 2: Write one brief, use it for every auditor

The brief must be identical for all auditors so differences in findings come from the model, not the prompt. Include:

1. The role: "fresh-eyes auditor; find flaws, failure modes, silent-failure paths, security weaknesses, data-loss risks, and wrong assumptions the author may not see; Swiss-cheese model: we don't know what we don't know."
2. The reading list, in order, with absolute paths, plus the claim under review.
3. Hard rules: read-only; do not edit any file; do not run anything that starts, stops, deletes or sends; never read, print or grep credential files, environment-variable files or private keys (name the project's ones explicitly); listing a file's permissions is allowed.
4. A checklist of angles specific to the work, written as questions. Generic angles worth including whenever they apply: can it be restored or rolled back, and is that written down; what deletes or corrupts data despite the safeguards; what fails silently (no log line, swallowed error, a "skipped" that hides lost coverage); concurrency and locking; the scheduled or automated run's environment (variables, mounts, permissions, notifications nobody sees); parsers and formats at their edges; growth with no cleanup policy; verification cadence; who else can reach it; whether the docs let a stranger operate it without this chat; and what is true today versus what the docs claim.
5. The output contract: a ranked list, most severe first; for each finding: severity (critical/high/medium/low), what exactly fails and how (a concrete scenario), evidence (file:line or quote), the smallest concrete fix; then a short "checked and found fine" list. Terse; no praise; a line cap (around 60).

## Step 3: Spawn the auditors in parallel

Use the Agent tool, `subagent_type: general-purpose`, fresh context (not a fork: the point is that they haven't seen the session). Launch all of them in one message so they run concurrently. Two is the default; three when the work is large or high-stakes. Choose different models for different auditors (for example one `opus` and one `fable`), since different models miss different things. Name each agent by model so the hand-backs can be told apart.

Do not touch the same files while they run; wait for all reports. Do not predict or summarise a report that hasn't arrived.

## Step 4: Merge

When all reports are in, merge them into one list for the user:

- De-duplicate: findings that several auditors raised appear once, marked as agreed; that agreement is itself a signal.
- Re-rank by real consequence for this user, not by the auditors' labels; drop or demote findings that are wrong on the facts, and say why in one clause.
- Split into three groups, with your own one-line take per item:
  - **Must fix**: concrete, consequential, and clear how. State who does it (you, the user, another session) and whether it can be done now or has to wait for something running.
  - **Discuss**: a judgement call with a trade-off. Give a recommendation and the cost of each option in one line each.
  - **Accept as known**: real but not worth the cost now; write these down so they aren't re-litigated.
- End with one question: go ahead with the must-fix list?

Keep it dense; one short paragraph or bullet per item. The user reads this to decide, not to admire the audit.

## Step 5: Record

After the user decides, write the outcome where the project keeps decisions (a `docs/decisions/` entry or the project's work log): the agreed fixes, the decisions on the discuss items with their reason, and the accepted-as-known list. The audit is only worth its cost if the next person can see what was decided and why.

## Do not

- Fork the current session as an auditor. A fork shares the blind spots; fresh context is the mechanism.
- Give auditors different briefs or let them edit. Differences must come from the model, and nothing may change under review.
- Paste the raw reports to the user. Merge, rank, and take a position.
- Fix things while the audit runs. Wait, merge, then act on what was agreed.
