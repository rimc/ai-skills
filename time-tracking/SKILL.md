---
name: time-tracking
description: "ALWAYS use this skill when the user says 'thanks', 'perfect', 'that worked', 'looks good', 'all set', 'done', 'ship it', 'nice', 'awesome', 'great', 'that's it', 'we're done', 'it works', or any short positive acknowledgment that isn't immediately followed by a new request in the same message. Also ALWAYS use when the user says 'log my time', 'track this task', 'track time', '/track-time', or asks about time tracking history or hours saved. Use when the conversation topic shifts to a clearly different task. This skill tracks time spent on coding tasks and estimates hours saved using Windsurf."
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash(mkdir:*)
  - Bash(echo:*)
  - Bash(cat:*)
  - Bash(date:*)
  - Bash(tail:*)
  - Bash(head:*)
  - Bash(wc:*)
  - Bash(grep:*)
---

# Time Tracking Skill

Automatically detect task completion, produce a time tracking summary with estimated hours saved, and persist entries to a monthly markdown log.

## Core Principle

Balance automatic detection with user control. Cascade proposes all values — the user confirms or adjusts before anything is written to disk.

## When to Activate

Activate when any of these conditions are met:

1. **Task completion signals** — The user says something like "that worked", "perfect", "all set", "we're done", "looks good", or otherwise signals a task is finished.
2. **Explicit request** — The user says "log my time", "track this task", "track time", or "/track-time".
3. **Topic shift** — The user's message is clearly about a different project, feature, or problem domain than what was just worked on.
4. **Break signals** — The user says "brb", "stepping away", "taking a break", etc. (note the timestamp; don't generate a summary yet — wait for the task to actually conclude).

**False positive guard:** If a short positive phrase like "thanks" or "perfect" is immediately followed by a new request *in the same message* (e.g., "thanks, now can you also..."), treat it as a continuation. But if the positive phrase is the entire message or ends the message, trigger the summary. When in doubt, trigger — the user can easily say to update a previous entry if the trigger was premature.

## Summary Output Format

When a task concludes, first write the entry to the timetracking file, then show the summary to the user. This way the time is logged even if the user walks away before responding.

**Sequence:**

1. Calculate all values (time, estimates, description)
2. Write the entry to the monthly timetracking file
3. Show the summary to the user:

```
## Time Tracking Summary

- **Task:** {project-name}: {Brief description of what was accomplished}
- **Expected Hours w/o Windsurf:** {X.X} hrs (my estimate — confirm or adjust)
- **Estimated Hours w/ Windsurf:** {X.X} hrs
  - First message: {HH:MM AM/PM} | Last message: {HH:MM AM/PM}
  - Excluded gaps: {X.X} hrs ({itemized list})
  - Declared breaks: {X.X} hrs ({itemized list, or "none"})
- **Hours Saved:** {X.X} hrs
- **Date:** {MM/DD/YYYY}
```

4. Ask: "This has been logged. Want to adjust anything? I'll update the entry if so."
5. If the user provides corrections, update the existing entry in the file.

## Time Calculations

### Estimated Hours w/ Windsurf

**Base:** Last message timestamp minus first message timestamp.

**Subtract idle time using three layered methods:**

1. **Explicit breaks** — When the user says "brb", "stepping away", "taking a break", etc., note the timestamp. When they say "back", "I'm back", "resuming", etc., note the return. Subtract the gap. If the user never explicitly returns but sends a new message after a gap, treat that message as the implicit return.

2. **Automatic gap detection** — Any gap longer than 30 minutes between consecutive messages is flagged as likely idle time and subtracted.

3. **Evening/overnight sensitivity** — Gaps after 5:00 PM local time carry extra weight. A 20-minute gap at 2:00 PM may be thinking time; a 20-minute gap at 6:00 PM is more likely the user stepped away. Overnight gaps (e.g., last message 5:30 PM, next message 9:00 AM) are excluded entirely. Use the system's local time.

Itemize all excluded time in the summary so the user can review and override.

### Expected Hours w/o Windsurf

Propose an estimate based on:

- **Task complexity** — Scope of what was built, modified, or debugged
- **Research/troubleshooting factor** — Use the conversation itself as evidence. If it involved debugging an obscure error, estimate the Stack Overflow browsing, docs reading, and trial-and-error that would have happened manually.
- **Domain difficulty** — How specialized or niche the problem domain is

Present the estimate with 1–2 sentences of reasoning, then ask the user to confirm or adjust.

### Hours Saved

Simple subtraction: `Expected Hours w/o Windsurf` minus `Estimated Hours w/ Windsurf`.

If the result is negative (user adjusted Expected Hours to less than Estimated Hours), record it as-is. Hours Saved can be negative — don't editorialize.

## Task Description Generation

Propose both fields:

- **project_name** — Infer from workspace/repo name, conversation context, or the nature of the work
- **description** — Brief summary of what was accomplished, generated from the conversation (keep under 200 words)

Both are presented for confirmation before writing.

## File Persistence

### Path and Structure

Monthly files at `~/.codeium/windsurf/timetracking/YYYY-MM-timetracking.csv`.

File format (standard CSV — quote any values containing commas):

```csv
Task #,Task Description,Expected Hours w/o Windsurf,Estimated Hours w/ Windsurf,Hours Saved,Date
1,"project-name: Built auth flow with JWT refresh tokens",6.0,1.5,4.5,03/29/2026
2,"project-name: Fixed CSS grid layout bug on dashboard",1.5,0.3,1.2,03/31/2026
```

Always wrap the Task Description field in double quotes since it commonly contains commas. If a description contains double quotes, escape them by doubling (`"""`).

### Writing Entries

Follow this sequence:

1. Check if `~/.codeium/windsurf/timetracking/` exists. If not, create it with `mkdir -p`.
2. Check if the file for the current month exists (e.g., `2026-03-timetracking.csv`). If not, create it with the header row.
3. If the file exists, read the last entry to determine the next Task #, then append the new row.
4. If updating an existing entry (same task continued after a summary), find the row by Task # and replace it.

Use native Cascade file operations (Read/Write/Edit) first. If they fail, fall back to shell commands (`mkdir -p`, `echo >>`, `cat`). If shell commands also fail, display the entry in chat so the user can manually save it.

## Task Boundary Detection

### 1. Explicit Completion (highest confidence)

Trigger on any short positive acknowledgment: "thanks", "perfect", "that worked", "it works", "we're done", "all set", "that's it", "ship it", "nice", "awesome", "great", "looks good", "done", and similar.

**Only suppress** if the same message continues with a new request (e.g., "thanks, now can you also..."). A standalone "thanks" or "perfect" is a trigger. When in doubt, trigger — false positives are low-cost since the user can redirect.

### 2. Topic Shift (medium confidence)

If the user's next message is clearly about a different project, feature, or problem domain, ask before triggering:

> "It looks like we're shifting to a new task. Want me to log the previous task before we continue?"

Do NOT auto-generate the summary on topic shift — ask first.

### 3. Manual Trigger (explicit, always honored)

User says "/track-time", "log my time", or "track this task". This always triggers the summary immediately, no questions asked.

## Task Continuation

### Ambiguous cases — ask

If it's not clear whether the current work is a continuation of the previous task or a new task, ask:

> "Is this a continuation of the previous task, or should I start a new time entry?"

Don't guess. The user knows best.

### Same task continues after summary

If the user continues working on the same task after a summary was logged:

- Silently continue tracking timestamps and breaks
- When the continuation concludes, produce an **updated** summary
- Overwrite the existing row in the markdown file (same Task #)
- Tell the user it's an update, not a new entry

### New task after summary

- Treat as a fresh task with a new Task #
- Independent tracking begins from the first message of the new topic

## Edge Cases

- **User says "brb" but never says "back":** The next message after the gap is the implicit return.
- **Very short conversations (< 5 minutes):** Still track. Even quick answers save research time.
- **Multiple tasks in one conversation:** Each task gets its own entry. Topic shift detection handles boundaries.
- **Negative Hours Saved:** Allow it. Don't comment on it or try to explain it away.
- **First-time use:** Create directory and file silently. No onboarding ceremony.
- **File write failure:** Fall back to shell commands. If that also fails, display the full entry in chat so the user can copy it.
- **User asks about history:** Read the current month's file (or a specified month) and present the data.

## Terminal Execution Note

Ensure directory creation and file write commands (`mkdir -p`, `echo >>`) are whitelisted in Windsurf's Auto Execution Policy for uninterrupted flow. If not whitelisted, Cascade will need user approval for each shell command.
