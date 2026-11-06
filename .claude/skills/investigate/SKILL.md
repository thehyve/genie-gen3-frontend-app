---
name: investigate
description: Answer a question through an iterative, user-steered investigation (clarify, scout, research plan with approval gate, research rounds with a checkpoint after each) across this repo, its sibling repos, the web (including vendor best practices) and read-only AWS/Kubernetes state, growing a question tree in .claude/tmp/<slug>/ and ending in a report and, on request, a target architecture and a prompt for /deliver. TRIGGER when the user asks to "/investigate" something, to research or find out how or why something works or fails, to compare options before deciding, or to resume an investigation (`/investigate <slug>`).
argument-hint: "[<question> | <slug>]"
---

# Investigate: clarify → scout → plan → research ⟲ checkpoint → report → architecture → deliver prompt

You are the **lead**. You coordinate and synthesize; research runs in subagents with a fresh
context, and you do only small lookups yourself (reading an artifact, one `grep`). Subagents
write their own findings file and return a short summary; phases hand off through files, never
through pasted content.

An investigation answers questions. It never edits code or configuration outside its folder,
never commits, and never runs a mutating command (see Safety). It ends only when the user says
it is done.

## Artifacts

All artifacts live in `.claude/tmp/<slug>/`, named by a kebab-case `<slug>` derived from the
question. `.claude/tmp/` is git-ignored: artifacts are working documents and are never
committed. Durable knowledge goes into `CLAUDE.md` or `docs/` (see phase 9). Artifacts never
reference session scratchpad paths. File formats are in `templates.md` next to this skill; read
it before writing the first artifact.

| Artifact | Path | Written by |
| --- | --- | --- |
| Brief | `.claude/tmp/<slug>/brief.md` | lead |
| Scout notes | `.claude/tmp/<slug>/scout.md` | lead, from the scout subagents |
| Question tree | `.claude/tmp/<slug>/tree.md` | lead |
| Research plan | `.claude/tmp/<slug>/plan.md` | lead |
| Findings | `.claude/tmp/<slug>/findings/<node-id>.md` | research subagent (lead for `Explore`) |
| Round log | `.claude/tmp/<slug>/log.md` | lead |
| Report | `.claude/tmp/<slug>/report.md`, then `report-v2.md`, … | lead, from the `general-purpose` subagent's returned text |
| Architecture | `.claude/tmp/<slug>/architecture.md` | lead, from the `general-purpose` subagent's returned text |
| Deliver prompt | `.claude/tmp/<slug>/deliver-prompt.md` | lead, from the `general-purpose` subagent's returned text |

## Round log

`log.md` is the run log for a later retro and the memory that survives context truncation.
After every phase, append: what the phase covered, the nodes touched, and
- every user correction, re-clarification, rejection or interruption, quoted verbatim;
- every subagent result that was wrong, empty or duplicated, and why;
- every permission prompt that was denied or approved repeatedly;
- every instruction (`CLAUDE.md`, skill, memory) that was missing, wrong or not followed.

## Safety

Research subagents change no file other than their findings file and run read-only commands
only: `git log/show/diff/blame`, file reads and searches, web search and fetch,
`aws … describe-*/get-*/list-*` and CloudWatch Logs queries, `kubectl get/describe/logs`, and
`make -C ../genie-infra-aws/deployments/dev plan`. Never `tofu apply`, `make apply`, image
pushes, deploys, `git pull`/`commit`/`push`, or writes to AWS, AppSync, S3 or the cluster. The
AWS profile is `genie-hyve-admin-918475456291` (dev); another environment is used only when the
brief names it. When a session check fails, ask the user to run
`! aws sso login --profile genie-hyve-admin-918475456291`.

## Starting and resuming

- `/investigate <question>`: new investigation; go to phase 1.
- `/investigate <slug>` (an existing folder): resume. Read `brief.md`, `tree.md` and the tail of
  `log.md`; compare the commits recorded in the brief with each repo's `HEAD` and say which repos
  moved; then go to phase 5 and present the open, partial and parked nodes.
- `/investigate` without arguments: list the folders in `.claude/tmp/` that contain `tree.md`,
  each with the brief's question and the number of open nodes, and ask which to resume or
  whether to start a new one.

## Phases

### 1. Clarify (lead)

Follow "Clarify Before Acting" in `CLAUDE.md`, limited to what is needed to aim the scout:
- **intent**: the decision, change or explanation the answer feeds;
- **question type(s)**: codebase / infra, external tech, debugging / incident, decision / design;
- **scope**: repositories, environments and external systems in and out;
- **done**: what an answer must contain to count as answered.

Skip what the request already answers; ask at most one `AskUserQuestion` round here, since
questions of substance are better asked after the scout. Each option covers a single concern,
names code by its path and says what it is, and gives its consequence and why it is
recommended. An option that depends on tool behaviour (flags, hooks, runtime limits) says in
plain words what the user would type and see; when that does not fit in an option, the
comparison goes into a findings file or the report first and the question points to it. These
rules apply to every `AskUserQuestion` in this skill. Choose the slug and tell the user. Record each relevant repo's branch and short
`HEAD` commit, and write `brief.md`.

### 2. Scout (one or two subagents, capped)

A cheap, broad pass that grounds the plan; it maps the terrain and does not answer the question.
Launch, in a single message, an `Explore` subagent for code (this repo and the sibling repos in
scope) and a `general-purpose` subagent for the web. The web scout runs for every question
except a pure lookup of local state (what a file contains, what a deployed resource is set to):
even a question about this codebase usually has a vendor or community best practice that
reframes it.
Each has a budget of about 10 tool calls and is told to start with short, broad queries. Code
scouting covers where the relevant code, config and data live, the vocabulary used, and existing
write-ups: `docs/`, `CLAUDE.md` or `AGENTS.md` of each repo, `../genie-infra-aws/.claude/plans/`
and `.claude/designs/`, and earlier investigations in `.claude/tmp/`. Web scouting identifies the
authoritative sources (official documentation, upstream repositories, issues and PRs, release
notes) and searches for best practices for the problem: vendor engineering write-ups and
guides (Anthropic for agents, skills and prompts; AWS for services and architecture; OpenTofu,
Amplify or Gen3 for their tools), then established community practice. Best practices often
change the shape of the answer, not only its details: designing this skill, Anthropic's write-up
on its multi-agent research system supplied the cheap broad pass before planning, the files
that survive context truncation and the effort scaling per question, which became the scout,
the artifacts and the round budget. The scout records which practices apply and how they would
change the plan. Both return candidate sub-questions and the terms or names that turned out ambiguous.
Combine their output into `scout.md`.

### 3. Research plan (lead) — approval gate

Turn the brief and scout into `tree.md` (top-level questions from the brief, sub-questions from
the scout, dependencies between them) and `plan.md` for the first round. Per planned node, the
plan states the method, subagent type, sources, budget and the evidence that would answer it:

| Type | Subagent | Method |
| --- | --- | --- |
| codebase / infra | `Explore`, one per repo | paths with line numbers, call and data flow, git history for why |
| external | `general-purpose` | web search and fetch; official docs and upstream code over blogs |
| debugging | `general-purpose` | read-only commands from Safety, dev unless the brief says otherwise; reproduce locally on sample input where possible |
| decision / design | `general-purpose` | best practices first (vendor guidance, comparable designs in the sibling repos and upstream), then an options matrix: criteria, evidence per option, recommendation; uses answered nodes of the other types; for each constraint the options work around, establishes why it exists (`git log`, `git blame`) and includes an option that removes or narrows it |

A decision / design question, and any question about how something should be built, gets a
best-practice node in the first round when the scout did not already answer it, so later rounds
compare the options against that practice instead of against each other only.

Scale effort to the node: one subagent with a few tool calls for a lookup, 2–4 subagents for a
comparison. A round launches at most 4 subagents; nodes that depend on an unanswered node wait
for a later round. When a node needs more, the plan says so and why.

Present the tree and the plan in a few bullets, plus the ambiguities the scout surfaced as
`AskUserQuestion` questions (e.g. which of two components a term refers to). Restate a short or
ambiguous answer as the concrete change to the brief or tree and confirm it before applying.
**Stop until the user approves the plan.**

### 4. Research round (parallel subagents)

Launch the planned nodes of the round in a single message. Each prompt contains: the node id and
question, the path of `brief.md` to read first, the method, sources and budget from the plan, the
findings file to write (`.claude/tmp/<slug>/findings/<node-id>.md`, format in `templates.md`),
the Safety rules, and these evidence rules:
- every claim cites its evidence: `path:line` at the recorded commit, a URL with a short quote,
  or a command with an excerpt of its output;
- every claim is graded `verified` (seen directly), `inferred` (follows from verified claims) or
  `unverified` (stated by a source that was not checked against the code or system);
- conflicting evidence is recorded as a contradiction, not resolved by picking one side;
- questions that came up but are outside the node are listed as new questions, not researched.

Each subagent returns at most 150 words: the answer, its confidence, new questions and gaps.
An `Explore` subagent cannot write files: it returns its findings in the file format instead, and
the lead writes them to the findings file unchanged.

### 5. Synthesize and checkpoint (lead) — after every round

Update `tree.md`:
- set each researched node to `answered`, `partial` or `open`, linking its findings;
- insert new nodes from the findings' new questions and gaps, with the finding as origin;
- add a node per contradiction;
- set nodes that turned out irrelevant to `dropped`, with the reason.

Append the round to `log.md`. Present: what was learned (a few bullets, each with its
confidence), contradictions, and the nodes now open, then ask with `AskUserQuestion` how to
continue:
- **go deeper**: choose nodes for the next round; update `plan.md` with a new round section and
  return to phase 4;
- **re-clarify**: the user restates what they mean; update `brief.md` (with a revision entry),
  rephrase, re-parent or drop the affected nodes, re-scout when the scope changed, and return to
  phase 3;
- **add a question**: a new node, possibly a new top-level question; return to phase 3;
- **write the report**: phase 6;
- **write the architecture**: phase 7, once a report exists and the change is a design;
- **pause**: stop; `/investigate <slug>` resumes here.

Recommend the option that best serves the brief's definition of done, and say so when the
recent rounds stopped adding new answers.

### 6. Report (`general-purpose` subagent)

Claude Code refuses a subagent's write of a report-like file ("Subagents should return
findings as text, not write report files"), so the report, architecture and deliver-prompt
subagents write no file: each returns the full document as its answer, and the lead writes it
unchanged to its artifact path before summarizing it.

Prompt a `general-purpose` subagent, instructed to do no new research and to return only the
report, with the paths of `brief.md`, `tree.md` and `findings/`. The lead writes it to
`report.md` (on a reopened investigation the next `report-v<n>.md`). It follows the format of
`templates.md`: the answer
first, then per top-level question the answer, evidence and confidence, contradictions, open and
parked questions, and recommendations. The report follows "No Chat History in Written Output" in
`CLAUDE.md`. Summarize it for the user and return to phase 5: the investigation stays open until
the user declares it done.

### 7. Architecture (`general-purpose` subagent, then lead) — before the deliver prompt

When the investigation leads to a change in design or structure, write the target architecture
before the deliver prompt, so the user reviews and steers the design as a whole instead of as
report recommendations. Prompt a `general-purpose` subagent, instructed to do no new research,
with the paths of `brief.md`, the latest report and `findings/`, to return `architecture.md` in
the format of `templates.md`: goals, layers and dependencies, files with a one-line purpose
each, the runtime flow of one representative operation, each key mechanism with a short
illustrative code sketch, a before/after example of one representative piece of code, a
numbered table of every clean-up (location, change, reason), decided points and open decisions.
The lead writes it to `architecture.md` and presents a summary.

The user then steers the document: each layout change or correction is applied to
`architecture.md` and recorded as a revision in `brief.md`, and each open decision is asked as
its own `AskUserQuestion` question (up to four per call) and moved to "Decided" once answered.
`architecture.md` is the leading design wherever it differs from the report. It follows "No Chat
History in Written Output" in `CLAUDE.md`.

### 8. Deliver prompt (`general-purpose` subagent) — on request

When the user asks for it, prompt a `general-purpose` subagent with the paths of `brief.md`, the
latest report and, if written, `architecture.md` (leading where they differ) to return
`deliver-prompt.md` in the format of `templates.md`; the lead writes it. The prompt is
pasted as the goal of `/deliver`, so it stands on its own: it covers what `deliver`'s Clarify
phase asks (repositories, entry points by path, coarse in/out of scope), the behaviour that is
decided and its evidence, constraints, open decisions with options and a recommendation, and
the paths of the report, architecture and findings for `deliver`'s Research phase to read. Every
file, function and test the prompt names is checked to exist in the current checkout before the
lead writes it. It follows "No Chat History in Written Output" in `CLAUDE.md`. Show the prompt to
the user.

### 9. Done (lead)

When the user declares the investigation done:
- following the Self-Improvement section of `CLAUDE.md`, propose durable knowledge from the
  report for `CLAUDE.md` or `docs/` (or for `../genie-infra-aws/.claude/designs/` when it
  concerns that repo), and write only what the user accepts;
- offer to run the `retro` skill with `<slug>`, passing `log.md` as the run log.

## Final report

End with the paths of the latest report and, if written, `architecture.md` and
`deliver-prompt.md`, and the number of
nodes answered, partial, open and dropped.
