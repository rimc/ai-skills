---

name: time-tracking
description: "ALWAYS use this skill when the user says 'thanks', 'perfect', 'that worked', 'looks good', 'all set', 'done', 'ship it', 'nice', 'awesome', 'great', 'that's it', 'we're done', 'it works', or any short positive acknowledgment that isn't immediately followed by a new request in the same message. Also ALWAYS use when the user says 'log my time', 'track this task', 'track time', '/track-time', or asks about time tracking history or hours saved. Use when the conversation topic shifts to a clearly different task. This skill tracks time spent on coding tasks and estimates hours saved using Windsurf."
allowed-tools:  - Read
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

## Configuration

<!-- Set your name below. This will be included as the User column in every time tracking entry. -->

**User Name:** `Your Name Here`
**Idle Gap Threshold:** `30` minutes

Automatically detect task completion, produce a time tracking summary with estimated hours saved, and persist entries to a monthly markdown log.

## Task Boundary Detection

### 1. Explicit Completion (highest confidence)

Trigger on any short positive acknowledgment: "thanks", "perfect", "that worked", "it works", "we're done", "all set", "that's it", "ship it", "nice", "awesome", "great", "looks good", "done", and similar.

**Only suppress** if the same message continues with a new request (e.g., "thanks, now can you also..."). A standalone positive phrase is always a trigger. When in doubt, trigger — false positives are low-cost since the user can redirect.

### 2. Manual Trigger (explicit, always honored)

User says "/track-time", "log my time", "track this task", "log this", or "track this". Always triggers immediately, no questions asked.

### 3. Topic Shift (medium confidence — ask first)

If the user's next message is clearly about a different project, feature, or problem domain, ask before triggering:

> "It looks like we're shifting to a new task. Want me to log the previous task before we continue?"

### 4. Break Signals (note, don't trigger)

When the user says "brb", "stepping away", "taking a break", etc., note the timestamp but don't generate a summary. The next message after the gap is the implicit return if they never say "back" explicitly.

### Ambiguous Cases

If it's unclear whether new work is a continuation or a new task, ask:

> "Is this a continuation of the previous task, or should I start a new time entry?"

## Summary Output Format

When a task concludes, first write the entry to the timetracking file, then show the summary. This way the time is logged even if the user walks away.

**Sequence:**

1. Calculate all values (time, estimates, description)
2. Write the entry to the monthly timetracking file
3. Show the summary:

```
## Time Tracking Summary

- **Task:** {project-name}: {Brief description of what was accomplished}
- **Expected Hours w/o Windsurf:** {X.X} hrs
  - {1–2 sentence phase breakdown}
- **Estimated Hours w/ Windsurf:** {X.X} hrs
  - First message: {HH:MM AM/PM} | Last message: {HH:MM AM/PM}
  - Excluded gaps: {X.X} hrs ({itemized list})
  - Declared breaks: {X.X} hrs ({itemized list, or "none"})
- **Hours Saved:** {X.X} hrs
```

4. Ask: "This has been logged. Want to adjust anything?"
5. If the user provides corrections, update the existing entry in the file.

## Time Calculations

### Estimated Hours w/ Windsurf

**Base:** Last message of the current task minus first message of the current task. In multi-task conversations, each task block has its own start and end — use the boundaries established by task boundary detection, not the first/last message of the entire conversation.

**Subtract idle time using three layered methods:**

1. **Explicit breaks** — Track "brb"/"back" pairs. If the user never explicitly returns, the next message is the implicit return.
2. **Automatic gap detection** — Gaps longer than the **Idle Gap Threshold** (see Configuration) between consecutive messages are subtracted.
3. **Evening/overnight sensitivity** — Gaps after 5:00 PM carry extra weight. Overnight gaps (e.g., 5:30 PM → 9:00 AM) are excluded entirely. Use the system's local time.

Itemize all excluded time in the summary so the user can review and override.

### Expected Hours w/o Windsurf

Estimate by breaking the task into **phases** and calibrating each one using **conversation evidence**.

**Phase breakdown — estimate each separately:**

| Phase | How to estimate |
|---|---|
| **Research / docs** | Count times Cascade referenced docs, corrected a misunderstanding, or explained a concept. Each maps to ~10–30 min of manual searching. |
| **Implementation** | Count files created/modified and lines of meaningful code. Baseline: ~50–100 lines/hr manually. Boilerplate skews higher; novel logic skews lower. |
| **Debugging** | Count error-fix cycles (user reports problem → fix → test). Each maps to ~15–60 min manually. Obscure or poorly-documented errors get the higher end. |
| **Integration / config** | Estimate from the number of distinct tools, environments, or systems touched. |

**Conversation signals that increase the estimate:**

- **More back-and-forth iterations** → higher debugging time
- **More files touched** → more context-switching overhead manually
- **External references** (docs, APIs, config specs) → research the user would have done by hand
- **Scope pivots** (changed approach mid-task) → count the abandoned path as additional manual time

**Sum the phase estimates** for the total. Present with a brief phase-level breakdown so the user can adjust individual components, not just one number.

### Hours Saved

`Expected Hours w/o Windsurf` minus `Estimated Hours w/ Windsurf`. Can be negative — record as-is without comment.

## Task Description Generation

Propose both fields:

- **project\_name** — Infer from workspace/repo name, conversation context, or the nature of the work
- **description** — Brief summary of what was accomplished (under 200 words)

Both are presented for confirmation before writing.

## File Persistence

### Path and Structure

Monthly files at `~/.codeium/windsurf/skills/time-tracking/logs/YYYY-MM-timetracking.csv`.

```csv
S #,Task Description,Expected hours without using Windsurf (A),Actual hours using Windsurf (B),Hours saved by Windsurf – (A)-(B),Entered By,Entered On Date
1,"project-name: Built auth flow with JWT refresh tokens",6.0,1.5,4.5,Your Name Here,03/29/2026
2,"project-name: Fixed CSS grid layout bug on dashboard",1.5,0.3,1.2,Your Name Here,03/31/2026
```

Always wrap Task Description in double quotes. Escape internal double quotes by doubling (`""`). Populate **Entered By** from the User Name config above.

### Writing Entries

0. **Check User Name config** — If still `Your Name Here`, ask the user and update the config before proceeding.
1. Ensure `~/.codeium/windsurf/skills/time-tracking/logs/` exists (`mkdir -p`).
2. Create the current month's file with header row if it doesn't exist.
3. Read the last entry to determine next S #, then append.
4. If updating an existing entry (task continued after summary), find the row by S # and replace it.

Use native Cascade file operations first. Fall back to shell commands. If both fail, display the entry in chat for manual copy.

## Task Continuation

**Same task continues after summary:** Silently keep tracking. On conclusion, produce an **updated** summary and overwrite the existing CSV row (same S #). Tell the user it's an update.

**New task after summary:** Fresh S # and independent tracking from the first message.

## Edge Cases

- **Short conversations (< 5 min):** Still track — even quick answers save research time.
- **Multiple tasks in one conversation:** Each gets its own entry via boundary detection.
- **Negative Hours Saved:** Allowed. Don't editorialize.
- **First-time use:** Create directory and file silently.
- **User asks about history:** Read the specified month's file and present the data.

## Terminal Execution Note

Ensure `mkdir -p` and `echo >>` are whitelisted in Windsurf's Auto Execution Policy for uninterrupted flow.
