# Huginn and Muninn — design

The contract document for the scout role. [`README.md`](README.md) carries the
myth and the reasoning; this file defines what a scout is sent, what it sends
back, and how it knows it is finished.

Everything here follows from one fact: **a scout starts cold.** It does not
inherit the orchestrator's conversation with the user, the prior turns, or
anything established in an earlier delegation. It knows the repository and the
brief, and nothing else. A brief that assumes shared context is not a thin
brief — it is a broken one, and the failure shows up as a confidently wrong
answer to a question the dispatcher never actually asked.

## Responsibility

Establish what is currently true in a repository, in answer to one stated
question, within one stated breadth — and report both the answer and the
boundaries of the search that produced it.

The scout owns three things and no others:

1. **Coverage.** Sweeping the territory the brief names, with enough search
   strategies that a thing hiding under an unexpected name is still found.
2. **Conclusions.** Claims with evidence, at the altitude the question was
   asked. Not the material the claims were derived from.
3. **Negative space.** Where it looked, what it did not cover, and how sure it
   is. This is the half that gets dropped, and it is the half Odin fears for.

The scout does not decide, plan, recommend, prioritize, or edit. It does not
adjudicate between its own findings and another scout's. See
[Read-only boundary](#read-only-boundary).

## Unit of dispatch

**The contract is per-scout: one brief, one question, one report, one thread.**
A multi-question brief is not supported, and decomposition is the dispatcher's
job.

Three reasons, in order of weight:

- **One breadth cannot serve several questions.** Breadth and budget are
  properties of a search, not of a dispatcher's curiosity. Two questions with
  different natural territories share one budget, so one of them gets
  over-searched and the other gets cut short — and the report cannot say which.
- **The done definition is per-question.** A compound brief produces one
  outcome value spanning parts that settled differently, which destroys
  information at exactly the point the system is most fragile.
- **Decomposition needs context the scout does not have.** Only the dispatcher
  knows why it is asking, so only the dispatcher can split a question into parts
  that are independently worth answering. A scout splitting its own brief is
  guessing.

Fan-out is therefore N briefs sent in parallel, sharing a `dispatch_id` prefix.
Two ravens fly separately and speak separately; Odin is the one who puts the two
reports together.

### Reconciling parallel scouts

Reconciliation is always the dispatcher's job. A scout never sees a sibling's
report and never adjusts its own findings to fit one.

| Situation | Resolution |
| --- | --- |
| **Overlap** — two scouts report the same thing | Dispatcher dedupes on evidence refs. Identical `path:line` refs are the same finding, not two. |
| **Gap** — neither scout covered something | Visible because every report's `coverage.not_searched` is mandatory. Dispatcher issues a follow-up brief for the uncovered strip. |
| **Contradiction** — two scouts assert incompatible facts | Dispatcher compares the evidence, not the confidence. If the evidence itself conflicts or is too thin to settle, issue a **tiebreak brief**: one `spot` or `focused` scout whose `question` names both claims verbatim and whose `given` carries both evidence sets. |

Confidence is a weighting signal, not a vote. A `high` does not beat a `low`;
evidence beats evidence. And a scout that adjudicates a contradiction has
started deciding, which is the one thing the role exists to avoid.

## Input contract — the brief

Required fields are required because the scout cannot obtain them by looking. A
missing one produces `blocked`, never a guess.

```
# --- identity and routing ---
dispatch_id: string                    (required)
  # Unique per scout thread; share a prefix across a fan-out (S1-auth-a, S1-auth-b).
  # Without it, parallel reports cannot be matched to the briefs that produced them.
dispatched_by: odin | mimir | sons-of-ivaldi | heimdall   (required)
  # Who to answer, and what the answer feeds. Echoed back; not a permission grant.

# --- where to look ---
repo_path: absolute path               (required)
  # The scout has no cwd it can trust and no way to infer the right checkout.
branch: string                         (required)
revision: commit sha (short ok)        (required)
  # Pins the report to a tree state. Without it, a report that contradicts a later
  # one is unresolvable — nobody can tell whether the code changed underneath.

# --- what is being asked ---
question: one interrogative sentence   (required)
  # Must be answerable by facts in the repository. If it cannot be phrased as a
  # question, it is not a scouting task — see Failure and edge cases.
sub_questions: list of strings, max 3  (optional)
  # Parts of the same question that must each be settled. Not separate questions;
  # separate questions get separate briefs.
decision_context: 1-3 sentences        (required)
  # What the answer will be used for, and what turns on it. This is the field that
  # decides relevance: without it the scout cannot tell a load-bearing detail from
  # trivia, and it cannot use the `incidental` channel at all. It is also the field
  # dispatchers omit most often, and the top cause of a technically correct,
  # useless report.
answer_shape: string                   (optional)
  # The form that makes the answer usable: "yes/no plus evidence", "a list of call
  # sites", "the one file that owns this". Saves a round trip when a question admits
  # answers of very different shapes.

# --- how wide to range ---
breadth: spot | focused | repo_wide | exhaustive   (required)
  # See the breadth table. Sets the stopping rule, the effort ceiling, and how
  # strong a negative claim the report is allowed to make.
territory: list of paths or globs       (required when breadth = spot or focused)
  # The boundary. Optional and advisory for repo_wide/exhaustive.
budget: { tool_calls: int, files_read: int }       (optional)
  # Overrides the breadth defaults. Use when the ceiling is known to be wrong.

# --- what the scout cannot know ---
given: list of statements               (required, may be "none")
  # Facts the dispatcher holds and the scout should treat as true without
  # re-deriving: the symptom, the error text, what the user actually said, what a
  # previous scout established. Cold start means the scout otherwise spends its
  # budget rediscovering things already known.
already_ruled_out: list of statements   (required, may be "none")
  # Hypotheses and locations already eliminated, with what eliminated them.
  # Must be written explicitly, including "none" — an omitted field reads as
  # "nothing ruled out" and an empty one reads as an oversight; the scout cannot
  # tell them apart.
vocabulary: list of term -> meaning     (optional)
  # User- or domain-language mapped to what it is likely called in the code
  # ("the council" = the five roles under agents/). A scout that greps the user's
  # word for a thing the code names differently finds nothing and reports absence.
siblings: list of "id: what it covers"  (optional)
  # What other scouts in this fan-out are covering. Lets this scout skip that
  # ground without recording it as a gap, and keeps two ravens off one branch.

# --- execution ---
may_run: list of exact commands         (optional, default empty)
  # Explicit grant to execute non-mutating project commands (a test, a build, a
  # CLI --help). Empty means read-only inspection only. Read-only git is always
  # permitted and does not need listing here.
```

### Breadth

Breadth is a discrete value with a defined stopping rule, not an adjective. The
effort ceilings are defaults; `budget` overrides them.

| `breadth` | Territory | Stop when | Ceiling | Strongest negative claim licensed |
| --- | --- | --- | --- | --- |
| `spot` | Exactly the locations in `territory` | Those locations have been read | ~5 files / ~8 calls | "Not present at the named locations." |
| `focused` | The paths in `territory`, plus files they import or name directly | Every file in territory is read or excluded by name, **and** two independent strategies over it surface nothing new | ~30 files / ~40 calls | "Not present in `<territory>`." |
| `repo_wide` | The whole working tree at `revision` | Three independent strategies (identifier grep, filename glob, string/config grep) each stop surfacing new locations | ~80 files / ~100 calls | "Not present in the working tree at `<revision>`." |
| `exhaustive` | Working tree, plus git history, CI/config, generated, vendored, and dependency manifests | `repo_wide` is satisfied **and** history and dependency surfaces are swept | ~150 files / ~200 calls | "Not present, and not removed by any commit reachable from `<revision>`." |

"Independent strategies" means the searches would fail differently: grepping an
identifier, globbing filenames, and grepping config or user-facing strings are
three. Grepping the same word three ways is one.

Early exit is licensed only by an affirmative answer: if the question is settled
with evidence before the stopping rule is met, stop and set
`coverage.stopped_early: true`. **Absence never licenses early exit** — an
unfinished sweep that found nothing is `partial`, not `exhausted_absent`.

### Brief validity

A brief is executable when every required field is present and `question` has a
single reading. Two readings that would drive the *same* search are not
ambiguity: proceed and record the reading in `interpretation`. Two readings that
would drive *different* searches are ambiguity: return `blocked`.

## Output contract — the report

The report has four bands, and the delineation between them is load-bearing:

| Band | What it is | Binding? |
| --- | --- | --- |
| `answer` | The conclusion, in the form `answer_shape` asked for | Yes — this is what the question bought |
| `findings` | Individual claims, each with evidence | Yes — everything in `answer` must trace to one |
| `coverage` / `unknowns` | The negative space: reach, gaps, confidence | Yes — this is how the dispatcher calibrates the rest |
| `incidental` | Observations outside the question | No — never changes the outcome, never gates done |

```
# --- identity ---
dispatch_id: string                     (required)   # echo of the brief
scout_id: string                        (required)   # which raven, in a fan-out
revision: commit sha + clean/dirty      (required)
  # Echo of the tree actually inspected. Makes a stale report detectable instead
  # of merely wrong.

# --- verdict ---
outcome: answered | partial | exhausted_absent | blocked    (required)
answer: prose, <= 120 words             (required, except when blocked)
  # Direct response to `question`, in `answer_shape`. Required even when the finding
  # is an absence — "No, and here is the boundary of that no" is an answer.
interpretation: 1 sentence              (required if the question admitted more
                                         than one reading and the scout proceeded)
blocked_on: string                      (required iff outcome = blocked)
missing_fields: list                    (required iff blocked for a missing field)

# --- claims ---
findings:                               (required unless blocked; max 7)
  - claim: one sentence, <= 40 words, indicative mood
      # What is true. Not what should be done about it.
    evidence: 1-4 refs                  (required)
      # file ref:   path/to/file.ext:12   or   path:12-24   [+ quote <= 2 lines]
      # search ref: <pattern or command> over <territory> -> <n> hits
      # Presence claims need >= 1 file ref. Absence claims need >= 1 search ref:
      # an unevidenced negative is indistinguishable from not having looked.
    confidence: high | medium | low     (required)
      # high   — read the code that does it
      # medium — inferred from two or more consistent indirect signals
      # low    — one indirect signal, or an inference across a gap
    contradicts_brief: true             (optional; set when the claim contradicts
                                         `given` or `already_ruled_out`)

# --- negative space ---
coverage:                               (required, including when blocked)
  strategies: list                      (required, >= 1)
    # How you looked, not only where. The dispatcher judges a negative result by
    # the strategies that produced it.
  searched: list of paths/globs         (required)
  not_searched: list of "what — why"    (required)
    # In-scope ground that was skipped, and why (budget, sibling coverage, judged
    # irrelevant). If nothing was skipped, write "none — full territory swept".
    # An empty value is a defect, not a clean sweep: a silent omission is invisible
    # in a way a wrong answer is not.
  stopped_early: true | false           (required)
  stop_reason: string                   (required)
    # stopping rule met | answer found | budget exhausted | blocked
unknowns: list, max 5                   (required, may be "none")
  # Questions raised and not settled, and what would settle each.

# --- out of scope ---
incidental: list, max 3                 (optional)
  - observation: indicative mood, with evidence
    relevance: one clause tying it to `decision_context`
  # Facts noticed outside the question that bear on what the dispatcher said it was
  # deciding. Stated as fact plus relevance — never as a recommendation.

effort: { tool_calls: int, files_read: int }   (optional)
  # Lets the dispatcher calibrate trust and tune later budgets.
```

### Outcome values

| `outcome` | Means | Also required | Dispatcher's move |
| --- | --- | --- | --- |
| `answered` | Coverage complete (or early exit licensed), every part settled, at least one part settled by something found | >= 1 finding | Use it |
| `partial` | Coverage incomplete, or >= 1 part unsettled | `not_searched` or `unknowns` names what is missing | Re-dispatch narrowed, raise `budget`, or accept |
| `exhausted_absent` | Coverage complete for the question's scope; the thing asked about is not there | >= 1 finding with search evidence | Treat the absence as fact at `revision` |
| `blocked` | The brief could not be executed as written | `blocked_on` | Fix the brief and re-dispatch |

Evaluate in order and take the first that matches:

1. The brief could not be executed → `blocked`.
2. The stopping rule was not met, or a part is unsettled → `partial`.
3. No part of the question was settled affirmatively → `exhausted_absent`.
4. Otherwise → `answered`.

`exhausted_absent` additionally requires that the negative claim licensed by
`breadth` covers the scope of the question. A repo-scoped question searched at
`focused` breadth cannot be `exhausted_absent`; it is `partial`.

A mixed result — some parts found, some shown absent, coverage complete — is
`answered`, with the absences stated in `answer` and carried by search-evidence
findings.

### Volume

The dispatcher shares the filesystem. It can read any file the scout read. What
it cannot do without the raven is know *which* file, *which* line, and that the
rest of the sweep came up empty. So the report ships location and claim, never
bytes.

| Cap | Value | On breach |
| --- | --- | --- |
| Findings | 7 | Merge to the altitude the question was asked at; residue to `unknowns` |
| Claim length | 40 words | Split the claim or raise the altitude |
| `answer` | 120 words | Cut to the conclusion; detail belongs in findings |
| Quoted lines | 20 total, 5 per quote | Cite `path:start-end` instead |
| `incidental` | 3 | Keep the ones tied to `decision_context`; drop the rest |
| Report total | ~500 words | Compress — never truncate silently |

Prohibited outright: pasted file contents, directory listings as output, tool
transcripts, "here is what I read" narration. Needing more than seven findings
is itself a finding — the question was compound. Say so in `unknowns`.

## Read-only boundary

Enforced three ways.

**Tools.** Permitted: reading, listing, globbing, grepping, and read-only git
(`log`, `show`, `diff`, `blame`, `ls-files`). Forbidden: any write or edit, any
git command that touches refs, index, or worktree, any install or fetch, and
running project code — unless the exact command appears in `may_run`.

**Mood.** Findings are written in the indicative. "X is", "X does", "X does not
exist". Banned constructions: *should*, *recommend*, *suggest*, *we need to*,
*the fix is*, *better to*. A claim that cannot be phrased as a statement of what
is true is not a finding — it is a decision wearing one.

**The incidental channel.** A scout will notice things that are out of scope and
genuinely matter, and it still has to surface them without becoming the decider.
The mechanism is `incidental`: state the observation as fact with evidence, then
one clause connecting it to the dispatcher's stated `decision_context`. The
scout supplies the fact and the link; the dispatcher supplies the judgment.

> Wrong: "The retry logic here is missing a timeout and should be fixed."
> Right: "`client.py:40` calls `send()` with no timeout, unlike the other three
> call sites. Relevant because you said you are deciding whether the hang is in
> this path."

`incidental` items never change `outcome` and never gate done. That is what
keeps the channel from becoming a second report.

## Done definition

Done is not "found the answer." A scout that swept exhaustively and found
nothing has succeeded — the absence is the finding, and it is often the most
expensive fact in the system to establish. Done is *the question is settled or
provably unsettleable within the brief, and the reach of the search is on
record.*

The scout self-evaluates against the brief it holds. All five must hold:

| # | Check | Derived from | If it fails |
| --- | --- | --- | --- |
| 1 | Every required brief field is present and `question` has one reading | brief validity rules | `blocked` |
| 2 | The stopping rule for `breadth` is met, or an affirmative early exit, or the budget ran out | `breadth`, `territory`, `budget` | `partial` |
| 3 | Each part of `question` / `sub_questions` is settled affirmatively (file evidence), settled by absence (search evidence), or listed in `unknowns` with what would settle it | `question`, `sub_questions` | `partial` |
| 4 | The report validates: outcome matches the precedence rule, every finding carries evidence of the right kind, `coverage.not_searched` is non-empty, caps respected, no normative language | output contract | Fix before returning |
| 5 | `answer` answers `question` in `answer_shape`, and nothing in `answer` lacks a supporting finding | `question`, `answer_shape` | Fix before returning |

Two failure modes this rules out deliberately: **finding the answer without
recording coverage is not done** (the dispatcher cannot weigh an answer whose
reach is unknown), and **an unfinished sweep that found nothing is not
`exhausted_absent`** (it is `partial`, and reporting it otherwise converts a gap
into a false fact).

## Failure and edge-case handling

| Situation | Handling |
| --- | --- |
| A required field is missing | `blocked`, list `missing_fields`. Do not infer the value, do not proceed on a plausible default. |
| `question` has two readings implying different searches | `blocked`; state both readings in `blocked_on`. If the readings imply the same search, proceed and record `interpretation`. |
| A path in `territory` does not exist | `blocked`; name the path and any near-miss as fact. Do not silently substitute the file you think was meant. |
| Budget exhausted mid-sweep | Stop. `partial`. `not_searched` lists the remainder in the order you would have taken it, so a re-dispatch can resume rather than restart. |
| The answer requires running code and `may_run` is empty | `partial`; `unknowns` names the exact command that would settle it. |
| Evidence contradicts `given` or `already_ruled_out` | Report it as a finding with `contradicts_brief: true`, high in the list. Do not silently re-plan the search around it. The dispatcher owns its own premises. |
| A better question becomes obvious | `unknowns`, or `incidental` if it bears on `decision_context`. Do not chase it — the budget belongs to the question that was asked. |
| The answer lies outside the repository (dependency internals, runtime, network) | Report the boundary as a finding with search evidence, `exhausted_absent` scoped to the repo, and name where the answer would live. |
| The brief asks the scout to choose, rank, or recommend | `blocked`. A scout can establish facts about the options; it cannot pick one. `blocked_on` should propose the fact-shaped rewording. |
| A sibling's territory overlaps yours | Report your own findings only. Note the overlap in `not_searched` if you skipped ground on that basis. |
| The report would exceed its caps | Compress: merge findings, cite ranges instead of quoting. Never drop a `coverage` field to fit. |

## Worked example

A real question about this repository, at `cba09e3`.

### Brief

```
dispatch_id: S1-brief-schema
dispatched_by: odin
repo_path: /home/user/valhalla
branch: claude/agent-roles-scaffold-836v5f
revision: cba09e3

question: Does this repository define anywhere a field-level schema for the
  dispatch brief that an orchestrator sends to a sub-agent role?
decision_context: I am about to have Mímir plan a shared brief/report envelope
  covering all five roles. If a schema already exists I need to extend it rather
  than invent a competing one; if none exists, the plan starts from scratch.
  Anything that already constrains the shape of inter-role messages counts,
  including prose.
answer_shape: yes/no, plus the paths of the closest existing thing if no

breadth: exhaustive
territory: (whole repository)
budget: (defaults)

given:
  - The repo is documentation-only as far as I know; the five roles live under agents/.
  - Each role folder has a README.md.
already_ruled_out: none
vocabulary:
  - "brief" = the message a dispatcher sends to a sub-agent
  - "role" = one of the five agents named in agents/README.md
  - "the council" = the five roles collectively
siblings: none
may_run: none
```

### Report

```
dispatch_id: S1-brief-schema
scout_id: huginn
revision: cba09e3 (clean)
outcome: exhausted_absent

answer: No. Nothing in this repository defines fields, a schema, or any
  structured message format between roles. The closest thing is a one-line
  "**Inputs:**" / "**Outputs:**" prose pair in each of the five role READMEs,
  which names the kinds of things a role takes and returns but gives no field
  names, no types, and no required/optional distinction. There is no code, no
  config, and no template to extend.

findings:
  - claim: The repository contains 7 tracked files, all Markdown; there are no
      source, config, or template files at any path.
    evidence:
      - search: `git ls-files` over repo root -> 7 hits, all *.md
      - search: `git ls-files | grep -v '\.md$'` -> 0 hits
      - search: `ls -a` on repo root -> no hidden dirs but .git
    confidence: high
  - claim: No file mentions a schema, contract, field, envelope, or payload for
      messages between roles.
    evidence:
      - search: `grep -ril` for schema|contract|field|envelope|payload|json|yaml
        over all tracked files -> 0 hits each
    confidence: high
  - claim: Each of the five role READMEs states its inputs and outputs as one
      line of prose, not as named fields.
    evidence:
      - file: agents/odin/README.md:36-37
      - file: agents/huginn-and-muninn/README.md:36-39
      - file: agents/heimdall/README.md:40-42
      - search: `grep -rn '^\*\*Inputs:\*\*|^\*\*Outputs:\*\*'` -> 10 hits across
        all 5 role READMEs
    confidence: high
  - claim: "dispatch" appears three times, all prose describing that scouts are
      sent; none describes what is sent.
    evidence:
      - search: `grep -rn dispatch --include=*.md .` -> 3 hits: agents/README.md:43,
        agents/huginn-and-muninn/README.md:21 and :49
    confidence: high
  - claim: No schema was present earlier and removed; the entire history is three
      commits and no file has ever been deleted.
    evidence:
      - search: `git log --all --oneline` -> 3 commits
      - search: `git log --all --diff-filter=D --name-only` -> 0 hits
    confidence: high

coverage:
  strategies:
    - full enumeration via `git ls-files`, then read all 7 files end to end
    - case-insensitive keyword grep over all tracked content: schema, contract,
      field, brief, dispatch, payload, envelope, json, yaml
    - structural grep for the existing convention: ^\*\*Inputs:\*\*|^\*\*Outputs:\*\*
    - history sweep: `git log --all`, deleted-file filter, both branches
    - filesystem listing of repo root including hidden entries
  searched:
    - README.md, agents/README.md, and all five agents/*/README.md — read in full
    - repository root, including hidden entries
    - full commit history on main and claude/agent-roles-scaffold-836v5f
  not_searched:
    - .git internals beyond the log — no repository source lives there
    - remote branches other than origin/main and origin/<current> — those are the
      only two refs that exist
  stopped_early: false
  stop_reason: exhaustive stopping rule met — every tracked file read in full,
    history swept, no untracked or hidden source present

unknowns:
  - Whether such a schema exists outside this repository (a prior session's notes,
    a sibling repo). Nothing in the tree points to one, and this search cannot see
    past the repo. Settled by: naming the other location in a follow-up brief.

incidental:
  - observation: The five Inputs/Outputs lines are not parallel to each other —
    three name artifacts ("the diff, the plan", agents/heimdall/README.md:40), two
    name abstractions ("prior session context", agents/odin/README.md:36).
    relevance: they are the surface a shared envelope would have to line up with,
    and you said the plan is a shared envelope.
  - observation: agents/README.md:34 states scouts return "facts, to whoever
    asked" — the only statement anywhere about where a sub-agent's output routes.
    relevance: routing is a field the envelope would need and the repo has one
    sentence of intent on it.

effort: { tool_calls: 12, files_read: 7 }
```

Note what the report does not contain: the contents of the seven files. Odin can
read those. What Odin could not do without the raven is know that the sweep came
up empty, how far it reached, and where it stopped.

## Notes for the shared contract

Four more role specs are coming. These are the seams where a common envelope
will likely be extracted — flagged, not built.

| Almost certainly shared | Why |
| --- | --- |
| `dispatch_id`, `dispatched_by`, `repo_path`, `branch`, `revision` | Pure addressing and tree-pinning. Nothing about them is scout-specific. |
| `decision_context`, `given`, `already_ruled_out`, `vocabulary` | These exist because sub-agents start cold. Every role starts cold, so every role needs them. |
| `budget`, `may_run`, `effort` | Effort ceilings and execution grants apply to any delegated worker. |
| Evidence ref format (`path:line`, and search-with-hit-count) | Heimdall already anchors findings to file and line. One format across roles makes findings composable and dedupable. |
| `confidence` scale, `contradicts_brief` | Any role can be handed a false premise and needs a non-silent way to say so. |
| The `incidental` channel | The implementer README already describes exactly this behavior — surface the adjacent change, do not slip it in. Same mechanism, different role. |
| `blocked` and `partial` as outcome values, plus "missing field means blocked, never guess" | Underspecification is the shared failure mode. These two values and that rule should be verbatim across all five. |
| The five-check done shape (brief valid → work complete per stated stopping rule → each asked part settled or listed → report valid → answer usable) | The skeleton generalizes; only what fills each slot changes. |

| Looks scout-specific | Why, and what the analogue probably is |
| --- | --- |
| `breadth` and its stopping rules | The *slot* generalizes — every role needs a stated scope with a stopping rule — but the values do not. Heimdall's is diff scope; the implementer's is the plan's steps; Mímir's is depth of investigation. Expect a shared `scope` slot with role-specific enums. |
| `answered` / `exhausted_absent` | These are the scout's specialization of "settled". Heimdall's are pass/fail; the implementer's are built/blocked. Probably one shared enum with a role-specific tail rather than four disjoint sets. |
| `coverage` (searched / not_searched / stopped_early) | Strongest candidate for surprising generality — Heimdall wants "what of the diff I did not read" just as much. The implementer's analogue is steps-completed vs. steps-skipped, which is the same idea in different clothes. Worth designing as shared. |
| The indicative-mood rule | Scout-only, and deliberately. Mímir and Heimdall are *supposed* to be normative; forbidding "should" across all roles would break them. |
| `question` / `sub_questions` / `answer` / `answer_shape` | The scout's payload. Each role has its own — plan, diff, verdict. |
| Findings caps (7 / 40 words / 500) | Numbers are tuned to reconnaissance. The principle (conclusions, not transcripts) generalizes; the values should be re-derived per role. |

Open seam worth deciding early: whether `outcome` is one shared enum with
role-specific extensions or a per-role enum with two shared members
(`blocked`, `partial`). The orchestrator has to branch on it either way, and
that branch is the busiest piece of logic in the system.
