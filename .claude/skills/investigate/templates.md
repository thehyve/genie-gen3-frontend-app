# Investigate: artifact formats

Formats for the files in `.claude/tmp/<slug>/`. Keep headings as given so a resumed
investigation and `/deliver` can find their way around.

## brief.md

```markdown
# <question, one sentence>

- **Intent**: <the decision, change or explanation the answer feeds>
- **Type(s)**: codebase / infra | external | debugging | decision / design
- **Done when**: <what an answer must contain>

## Scope

- In: <repos, environments, external systems>
- Out: <…>

## Assumptions

- <assumption>, <why it is safe or what would change if wrong>

## Glossary

- **<term>**: <meaning in this investigation>

## Checkouts

| Repo | Branch | Commit |
| --- | --- | --- |
| genie-data-scripts | dev | abc1234 |

## Revisions

- r2: <what changed in the question or scope, and which nodes it affected>
```

## scout.md

```markdown
# Scout

## Where evidence lives

- `<path>`: <what it is (e.g. deployed Lambda handler, Amplify pipeline definition)>

## Existing write-ups

- `<path or URL>`: <what it covers, how current>

## Best practices

- <practice> — <source URL> (<vendor guide | engineering write-up | community>): <how it would
  change the plan or the answer>

## Vocabulary and ambiguities

- **<term>**: <meanings found, which components use it>

## Candidate sub-questions

- <question> (<why it matters for the brief>)
```

## tree.md

One line per node, indented by depth. Ids are stable: a dropped node keeps its id, a new child
of `Q1` gets the next free `Q1.<n>`, a contradiction gets `C<n>`.

```markdown
# Question tree

- **Q1** [answered] <question> — origin: brief — findings: [Q1](findings/Q1.md)
  - **Q1.1** [partial] <question> — origin: scout — depends on: — findings: [Q1.1](findings/Q1.1.md)
  - **Q1.2** [open] <question> — origin: Q1.1/F2
  - **Q1.3** [dropped] <question> — reason: <…>
- **Q2** [parked] <question> — origin: user, round 2
- **C1** [open] <claim A> vs <claim B> — origin: Q1/F1, Q2/F3
```

Statuses: `open`, `planned`, `answered`, `partial`, `parked` (deliberately postponed),
`dropped` (no longer relevant).

## plan.md

```markdown
# Research plan

## Round <n>

| Node | Type | Subagent | Sources | Budget | Answered by |
| --- | --- | --- | --- | --- | --- |
| Q1.1 | codebase | Explore (genie-data-scripts) | `src/genie_data/…` | ~10 calls | the code path from handler to write, with path:line |

Waiting: Q1.2 (depends on Q1.1).
```

Earlier rounds stay in the file, newest first.

## findings/<node-id>.md

```markdown
# <node-id>: <question>

**Answer**: <one or two sentences> (**confidence**: verified | inferred | unverified)

## Claims

- **F1** [verified] <claim>. Evidence: `src/genie_data/x.py:42` @abc1234
- **F2** [unverified] <claim>. Evidence: <URL> — "<short quote>"
- **F3** [inferred from F1, F2] <claim>.

## Contradictions

- <claim> vs <claim>, with both pieces of evidence.

## New questions

- <question> (raised by F2)

## Gaps

- <what could not be checked, and why>
```

## log.md

```markdown
# Log

## Round <n> — <phase>

- Covered: <nodes>
- Steering: "<user's words, verbatim>"
- Issues: <wrong or empty subagent results, permission prompts, missing instructions>
```

## report.md

```markdown
# <question>

## Answer

<direct answer to the brief's question, with overall confidence>

## Findings

### Q1: <question>

<answer, key evidence (path:line, URL), confidence>

## Contradictions

## Open and parked questions

## Best practices applied

- <practice> (<source>): <where it shaped the answer or recommendation>

## Recommendations

## Sources

- Findings: `.claude/tmp/<slug>/findings/`
- <URLs and paths used>
```

## architecture.md

The leading design for `/deliver`. Code sketches are illustrative and carry no comments.

```markdown
# <component>: target architecture

<one paragraph: what it covers; evidence in report.md and findings/>

## Goals

- <what the design must achieve, incl. "same result" constraints>

## Layers

<diagram of layers and dependency direction>

## Files

<tree with a one-line purpose per file; what is removed>

## <Runtime flow of one representative operation>

## <Key mechanism>, one section each

<explanation, table where useful, short code sketch>

## Example: before and after

## Clean-ups

| # | Item | Where | Change | Reason |
| --- | --- | --- | --- | --- |

Kept on purpose: <items and why>

## Decided

- <decision and rationale>

## Open decisions

1. <question>: <options>, <recommendation>
```

## deliver-prompt.md

Pasted verbatim as the goal of `/deliver`, so it is complete on its own.

```markdown
<Goal in one or two sentences.>

Repositories: <this repo / ../genie-infra-aws / ../genie-data-submission-portal>.
Entry points: `<path>` (<what it is>), …

In scope: <…>
Out of scope: <…>

Current behaviour (evidence): <behaviour> (`<path:line>`), …
Decided behaviour: <…>
Constraints: <deploy order, compatibility, permissions, …>
Best practices to follow: <practice> (<source URL>), …

Open decisions:
1. <question>: <option A> (recommended, because …) / <option B>.

Background: `.claude/tmp/<slug>/report.md` and `.claude/tmp/<slug>/findings/`.
```
