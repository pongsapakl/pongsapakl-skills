---
name: babysit
description: Watch a long-running job the session just started (a backup, upload, build, migration, seed, sync, training run) so the user doesn't have to. Polls often at first to prove the job is really progressing and not hung, then widens the cadence once it looks stable, reports one line per healthy check, flags a probable hang with numbers instead of killing it, honours a stop condition if one was given, and ends with a clear done/failed. No arguments needed: it watches whatever was just started. Auto-invokes on "babysit this", "keep an eye on it", "watch it", "check it every N minutes", "make sure it doesn't hang", "poll it and tell me when it's done", "wake me when it finishes".
allowed-tools: [Bash, Read, CronCreate, CronDelete, CronList, PushNotification]
---

# Babysit — adaptive watch over a long-running job

A job that was started a minute ago needs a different watch than one that has run cleanly for an hour. Early on, the question is "is this actually working?", and a 30-minute silence can hide a hang that wastes the whole night. Later, the question is only "is it still going?", and polling every 5 minutes is noise. This skill does the first kind of watching, then relaxes into the second.

It runs on the session's own scheduler (`CronCreate`), so it costs nothing while idle and survives the user walking away.

## What to watch

Usually clear from the conversation: the most recent long-running command, background task, or launchd/cron job the session started. If several candidates exist or none is obvious, ask one question with a recommendation, then proceed.

Identify, before the first check, how this particular job shows progress. Pick the signals that apply and write them into the check command, because "process exists" alone proves nothing:

- **process alive** (`pgrep` with an anchored pattern, so another process with a similar name can't fool the check)
- **CPU** (`ps -o etime=,%cpu= -p PID`) — a job doing I/O at low priority may legitimately sit at 1–5%, so read this together with the next signals
- **bytes or items moving** (`nettop -P -L 1 -J bytes_out -p PID` on macOS, a counter in the job's own log, a growing output file, a growing remote directory)
- **log advancing** (`tail -n 2` of the job's log and of any launchd/cron capture file)
- **error file empty** (if the job keeps one, its size must stay 0)
- **phase** (which stage the job is in, if the log says: scanning, uploading, verifying)

Record the baseline from the first check so later checks can say "moved" or "didn't move".

## Cadence

- **Start at 5 minutes** unless the user said otherwise.
- After the job has looked healthy for several checks in a row, **widen** to what fits the job: 15 minutes for something that finishes within hours, hourly for a multi-day run. Use judgment about the job's phases; a long read phase with no upload yet is normal for a backup, not a hang. If unsure whether a quiet stretch is normal for this job, ask the user rather than guess.
- Widen by deleting the old cron job and creating a new one; say the new cadence in one line.

## Hang rule

No progress on any signal for **two consecutive checks** (bytes flat, log unchanged, CPU near zero) → report it as a probable hang **with the numbers** (elapsed, CPU, bytes, last log line and its age). Never kill, restart, or clear locks on your own; the user decides. Exception: a stop condition the user gave in advance.

## Stop condition

If the user named one ("stop it before phase X", "kill it if it goes past 2 h", "stop before the big upload"), check for it every round and act on it exactly as stated, then say what was stopped and why in two lines.

## Reporting

- Healthy check: **one short line** (elapsed, the signals that moved, phase). Nothing else.
- Something changed (phase moved, cadence widened, error file grew, warning in the log): two or three lines.
- Done or failed: say which, with the final numbers (duration, amount processed, exit status, last log line), then delete the cron job. If the user may have walked away, send one push notification.

## Ending

Delete the cron job when the job is done, failed, stopped, or the user says stop. Never leave a watcher running on a job that no longer exists.

## Do not

- Report "running" without evidence of progress; a live PID is not progress
- Kill or restart the job on your own initiative
- Poll more often than the job's phase warrants once it has proven itself
- Pad healthy checks with recaps; one line
