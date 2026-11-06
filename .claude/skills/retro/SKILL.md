---
name: retro
description: Blameless retrospective of the current session or a /deliver run, turning corrections, failures and friction into concrete improvements of memory, CLAUDE.md, docs, skills, hooks or permissions, and logging them in docs/learnings.md. TRIGGER at the end of a /deliver run, when the user asks for "/retro", a retrospective, a post-mortem, or "what should we improve from this session".
argument-hint: "[slug]"
---

# Retro: learn from the work just done

A blameless review: the question is which instruction, tool or check was missing so that the same
problem does not happen again, not who made a mistake.

## 1. Collect facts

Use the run log passed by `/deliver`, or, when invoked standalone, review the current
conversation. Collect:
- user corrections, interruptions, rejected plans and reverted edits (quote the user's words);
- test or lint failures and their root causes;
- permission prompts that were denied or approved repeatedly;
- instructions in `CLAUDE.md`, skills or memory that were missing, wrong, outdated or ignored;
- things that went well and should be kept.

Read the run artifacts in `.claude/tmp/<slug>/` for the slug, if any.

## 2. Classify each finding

| Finding | Improvement target |
| --- | --- |
| How this user likes to work | auto-memory |
| Project convention or durable fact | `CLAUDE.md` (short) or `docs/` (long, with a pointer in `CLAUDE.md`) |
| Multi-step procedure that recurs | new or updated skill in `.claude/skills/` |
| Rule that must hold every time | hook in `.claude/settings.json` or a test |
| Repeated permission prompt | allowlist entry (suggest `/fewer-permission-prompts`) |
| Defect in a skill, including `deliver` or `retro` | edit to that `SKILL.md` |
| Memory entry that proved itself across several runs | promote into a checked-in file and remove it from memory |

Skip findings that are one-offs with no durable lesson.

## 3. Write the retrospective

Write `.claude/tmp/<slug>/retro.md` (use a date-based slug when standalone; the folder is
git-ignored) with sections:
- **Improvements**: one subsection per improvement, each with
  - **Problem**: what went wrong or was missing, with the facts that show it (quotes, failing
    test ids, error excerpts);
  - **Solution**: the target file and the exact proposed change;
  - **Discussion**: alternatives, trade-offs, risks and open points.
- **What went well**: practices to keep.
- **Rejected**: findings with no durable lesson, each with the reason.

Facts belong to the improvement they motivate, so each problem and its solution read together.

## 4. Propose and apply

Ask one `AskUserQuestion` question per improvement, up to four per call. Each question states
the problem in a sentence, the target file and the exact change; its options are apply,
alternatives from the Discussion where there are any, and skip, with the recommended option
first. Apply only the approved ones. All written changes follow "No Chat History in Written Output" in
`CLAUDE.md`: state the rule and its rationale, not the conversation that produced it.

## 5. Log

Append one line to `docs/learnings.md`:

```
- YYYY-MM-DD `<slug>`: <applied improvements, comma separated, with target file> (or "no changes")
```
