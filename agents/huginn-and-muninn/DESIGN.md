# Huginn and Muninn — design

The scout role has three files. [`README.md`](README.md) carries the myth and
the reasoning for the role's existence. [`brief.schema.json`](brief.schema.json)
and [`report.schema.json`](report.schema.json) are the contract — they are
normative, they are JSON Schema draft 2020-12, and every field carries its own
`description` explaining what it is and why it exists, so the documentation
cannot drift from the thing it documents. This file explains: it covers the
reasoning, the semantics a schema cannot encode, and the rules that span more
than one field or more than one document.

Read the schemas for *what*. Read this for *why*, and for the parts a validator
cannot check.

Everything here follows from one fact: **a scout starts cold.** It does not
inherit the orchestrator's conversation with the user, the prior turns, or
anything established in an earlier delegation. It has the brief and the
repository, and nothing else. A brief that assumes shared context is not terse,
it is broken — and the failure it produces is a confident, well-evidenced answer
to a question nobody asked.

## Responsibility

Establish what is currently true in a repository, in answer to one stated
question, within one stated breadth — and report both the answer and the
boundaries of the search that produced it.

The scout owns three things and no others:

1. **Coverage.** Sweeping the territory the brief names, with enough distinct
   search strategies that a thing hiding under an unexpected name is still
   found.
2. **Conclusions.** Claims with evidence, at the altitude the question was
   asked — not the material the claims were derived from.
3. **Negative space.** Where it looked, what it did not cover, how sure it is.
   This is the half that gets dropped, and it is the half Odin fears for.

The scout does not decide, plan, recommend, prioritize, or edit, and it does not
adjudicate between its own findings and another scout's.

## Unit of dispatch

**One brief, one question, one report, one thread.** The schema enforces the
shape — `question` is a single string, not an array — and decomposition is the
dispatcher's job.

- **One `breadth` cannot serve several questions.** Breadth and budget are
  properties of a search, not of a dispatcher's curiosity. Two questions with
  different natural territories share one ceiling, so one gets over-searched and
  the other gets cut short, and the report cannot say which.
- **`outcome` is per-question.** A compound brief yields one outcome value
  spanning parts that settled differently, destroying information exactly where
  this system is most fragile.
- **Decomposition needs context the scout does not have.** Only the dispatcher
  knows why it is asking, so only the dispatcher can split a question into parts
  independently worth answering. A scout splitting its own brief is guessing.

`sub_questions` exists for parts of one question that must all be settled before
it is answered — not as a back door to compound briefs. If two entries could be
dispatched to different scouts without loss, they are two questions.

Fan-out is therefore N briefs sent in parallel sharing a `dispatch_id` prefix.
Two ravens fly separately and speak separately; putting the two reports together
is Odin's job, not theirs.

### Reconciling parallel scouts

Reconciliation always belongs to the dispatcher. A scout never sees a sibling's
report and never adjusts its findings to fit one.

| Situation | Resolution |
| --- | --- |
| **Overlap** — two scouts report the same thing | Dedupe on evidence. Structured `file` evidence makes this mechanical: same `path` and overlapping line range is one finding, not two. |
| **Gap** — neither scout covered something | Visible because `coverage.not_searched` is mandatory and non-empty in every report. Dispatch a follow-up for the uncovered strip. |
| **Contradiction** — two scouts assert incompatible facts | Compare the evidence, not the `confidence`. If the evidence itself conflicts, or is too thin to settle, issue a **tiebreak brief**: one `spot` or `focused` scout whose `question` names both claims verbatim and whose `given` carries both evidence sets. |

`confidence` is a weighting signal, not a vote — a `high` does not beat a `low`,
evidence beats evidence. And a scout that adjudicates a contradiction has begun
deciding, which is the one thing the role exists to avoid. The `siblings` field
exists so a scout can *avoid* overlap, not so it can resolve it.

## What the brief must carry, and why

Every required field in `brief.schema.json` is required because the scout cannot
obtain it by looking. The groups:

- **Where to look** (`repo`). A cold thread has no working directory it can
  trust. The `revision` matters more than it looks: it pins every claim in the
  report to a tree state, so two reports that contradict each other can be
  resolved instead of merely disagreeing.
- **What is being asked** (`question`, `sub_questions`, `decision_context`,
  `answer_shape`). `decision_context` is the field that makes relevance
  judgeable. Without it the scout cannot separate a load-bearing detail from
  trivia and cannot use the `incidental` channel at all, since relevance is
  defined against that text. It is also the field dispatchers omit most often
  and the top cause of a correct, useless report.
- **How wide to range** (`breadth`, `territory`, `budget`). See below.
- **What the scout cannot know** (`given`, `already_ruled_out`, `vocabulary`,
  `siblings`). This is the cold-start repair kit. `vocabulary` is the quiet one:
  a scout that greps the user's word for a thing the code names differently
  finds nothing and reports an absence that is not real — the most dangerous
  output this role can produce.
- **Execution** (`may_run`). Default empty. See
  [Read-only boundary](#read-only-boundary).

`given` and `already_ruled_out` are required arrays that may be empty. That is
deliberate: making them required means an empty array unambiguously says
"nothing to give", where an absent field would be indistinguishable from a
dispatcher who forgot.

### Breadth

`breadth` is an enum in the schema; these semantics are what the enum values
mean, and they are binding. The effort ceilings are defaults that `budget`
overrides.

| `breadth` | Territory | Stop when | Default ceiling | Strongest negative claim licensed |
| --- | --- | --- | --- | --- |
| `spot` | Exactly the locations in `territory` | Those locations have been read | ~5 files / ~8 calls | "Not present at the named locations." |
| `focused` | `territory`, plus files it imports or names directly | Every file in territory is read or excluded by name, **and** two independent strategies over it surface nothing new | ~30 files / ~40 calls | "Not present in `territory`." |
| `repo_wide` | The whole working tree at `repo.revision` | Three independent strategies each stop surfacing new locations | ~80 files / ~100 calls | "Not present in the working tree at this revision." |
| `exhaustive` | Working tree, plus git history, CI and config, generated, vendored, dependency manifests | `repo_wide` is satisfied **and** history and dependency surfaces are swept | ~150 files / ~200 calls | "Not present, and not removed by any commit reachable from this revision." |

**Independent** means the searches would fail differently — grepping an
identifier, globbing filenames, and grepping user-facing strings are three.
Grepping the same word three ways is one. This is why
`coverage.strategies[].method` is an enum rather than free text: the dispatcher
can count distinct methods without reading prose, and the schema requires at
least three entries once `breadth` reaches `repo_wide`.

Early exit is licensed only by an affirmative answer. **Absence never licenses
early exit**: an unfinished sweep that found nothing is `partial`, never
`exhausted_absent`. The schema encodes this by requiring
`coverage.stop_reason` to be `answer_found` whenever `stopped_early` is true.

### Outcome

`answered` is not "found something good" and `exhausted_absent` is not a
failure. Evaluate in order, take the first that matches:

1. The brief could not be executed as written → `blocked`.
2. The stopping rule was not met, or any part of the question is unsettled →
   `partial`.
3. No part of the question was settled affirmatively → `exhausted_absent`.
4. Otherwise → `answered`.

| `outcome` | Dispatcher's move |
| --- | --- |
| `answered` | Use it. |
| `partial` | Re-dispatch narrowed, raise `budget`, or accept the gap knowingly. `coverage.not_searched` lists the remainder in the order a resumption should take it. |
| `exhausted_absent` | Treat the absence as fact *at that revision and that breadth*. This is frequently the most expensive fact in the system to establish. |
| `blocked` | Repair the brief and re-dispatch. `blocked_on.missing_fields` is machine-readable so the repair can be mechanical. |

A mixed result — some parts found, some shown absent, coverage complete — is
`answered`, with the absences stated in `answer` and carried by findings whose
`claim_kind` is `absence`.

## Cross-field rules

Rules that span fields or documents. The first block is enforced by the schemas;
the second cannot be, and is binding anyway.

**Encoded** (`if`/`then`, `contains`, `minItems`, `required`):

| Rule | Where |
| --- | --- |
| `breadth` of `spot` or `focused` requires `territory` | brief, top-level `if`/`then` |
| Any `outcome` but `blocked` requires `answer` and `findings` | report `allOf` |
| `outcome: blocked` requires `blocked_on` | report `allOf` |
| `outcome: exhausted_absent` requires at least one finding with `claim_kind: absence` | report `allOf`, via `contains` |
| A `presence` claim requires at least one `file` evidence ref; an `absence` claim requires at least one `search` ref | `$defs.finding.allOf` |
| `coverage.stopped_early: true` requires `stop_reason: answer_found` | report `allOf` |
| `breadth` of `repo_wide` or `exhaustive` requires ≥3 `coverage.strategies` | report `allOf` |
| `blocked_on.reason: missing_required_field` requires `missing_fields` | `blocked_on.if`/`then` |
| `coverage.not_searched` must have ≥1 entry | `minItems` |
| `given`, `already_ruled_out`, `unknowns` may be empty but may not be absent | `required` |

**Not encodable, still binding:**

| Rule | Why a schema cannot check it |
| --- | --- |
| `report.dispatch_id` equals `brief.dispatch_id`; `report.repo.revision` and `report.breadth` echo the brief | Spans two documents. A validator sees one at a time. |
| Nothing appears in `answer` that is not supported by a `findings` entry | Semantic. |
| The ≥3 strategies at wide breadth must use three *different* `method` values | JSON Schema counts items, not distinct property values. |
| `line_end` ≥ `line_start` | No cross-property numeric comparison. |
| `exhausted_absent` requires the negative claim licensed by `breadth` to cover the *scope of the question* — a repo-scoped question searched at `focused` breadth is `partial`, not absent | Requires understanding the question. |
| `interpretation` must be present whenever the question admitted more than one reading and the scout proceeded anyway | Requires understanding the question. |
| Claims are in the indicative mood; no recommendations | Semantic. See below. |
| Evidence paths must exist at `repo.revision` and say what the claim says they say | Requires the repository. |

The unencodable half is not decoration. A report that validates and violates
these is worse than one that fails validation, because it looks clean.

## Prose in JSON

JSON is the transport because it parses in a terminal and has explicit keys —
but findings are prose, and prose in JSON means escaped newlines, which are
miserable to write and worse to read.

**The call: every prose value is a single line with no embedded newline, capped
in length. Multi-sentence prose is an array of single-line strings.** Enforced
by `"pattern": "^[^\\n]+$"` plus `maxLength` on every prose field, which is why
`answer`, `decision_context`, and `blocked_on.detail` are arrays rather than
paragraphs.

Why this and not the alternatives:

- **Escaped multi-line strings** — `jq -r` renders them, but every other tool
  shows `\n` soup, and diffs of a changed report become one enormous line.
- **Embedded block markdown** — puts a second syntax inside the string that
  nothing in the pipeline parses, and invites the file-dump the role exists to
  prevent.
- **Arrays of lines** — `jq -r '.answer[]'` prints clean prose, `jq -r
  '.findings[].claim'` prints a scannable list, `grep` works on the output, and
  the length caps become enforceable per line rather than per blob.

Inline backticks for identifiers and paths are allowed — they survive `jq -r` as
plain text and are how a terminal reader already scans code references. Block
markdown (headings, fences, list markers) is not.

The caps are part of the design, not formatting fussiness. `claim` is 240
characters because a claim that needs more is two claims. `answer` is at most
three lines because the conclusion is not the report. `findings` is capped at
seven because needing more means the question was compound — merge to the
altitude the question was asked at and put the residue in `unknowns`.

## Evidence

Evidence is structured, not a `path:line` string, and it comes in exactly two
kinds discriminated by `kind`:

- **`file`** — `path`, `line_start`, optional `line_end`, optional two-line
  `quote`. Supports presence claims.
- **`search`** — `query`, `scope`, `hits`, optional `note`. Supports absence
  claims. `hits: 0` is the interesting value and the entire reason this kind
  exists.

Separate keys rather than one string, for three reasons: a consumer can open the
location without re-parsing; two scouts' evidence can be deduped on `path` and
line range mechanically; and `hits` as an integer lets the dispatcher filter for
searches that came up empty without pattern-matching English.

The `claim_kind` field on every finding is what makes the evidence rule
enforceable: the scout declares whether it is asserting existence or absence,
and the schema then requires the matching evidence kind. An unevidenced negative
is indistinguishable from not having looked, and that is the exact failure the
ravens exist to make impossible.

Prohibited outright, and not expressible in the schema: pasted file contents,
directory listings as output, tool transcripts, "here is what I read" narration.
The dispatcher shares the filesystem and can read anything the scout read. What
it cannot do without the scout is know *which* file, *which* line, and that the
rest of the sweep came up empty. The report ships pointers and claims; that is
the whole value add.

## Read-only boundary

Enforced three ways.

**Tools.** Permitted: reading, listing, globbing, grepping, and read-only git
(`log`, `show`, `diff`, `blame`, `ls-files`). Forbidden: any write or edit, any
git command touching refs, index, or worktree, any install or fetch, and running
project code — unless the exact command appears in the brief's `may_run`. That
field is a grant from the dispatcher, never a scout's own judgment call; a
`may_run` entry that would mutate anything is a defect in the brief and comes
back `blocked`. When the answer requires execution and no grant was given, the
outcome is `partial` and `unknowns[].would_settle` names the exact command.

**Mood.** Claims are written in the indicative: "X is", "X does", "X does not
exist". Banned constructions: *should*, *recommend*, *suggest*, *we need to*,
*the fix is*, *better to*. A claim that can only be phrased that way is a
decision wearing a finding's clothes. This is a lint over `findings[].claim` and
`incidental[].observation`, not something a validator can catch.

**The `incidental` channel.** A scout will notice things outside its question
that genuinely matter, and it still must not become the decider. The mechanism
is structural: `observation` states the fact in the indicative, `evidence`
supports it under the same rules as a finding, and `relevance` is one clause
tying it to the brief's `decision_context`. The scout supplies the fact and the
link; the dispatcher supplies the judgment.

> Wrong: "The retry logic here is missing a timeout and should be fixed."
> Right: observation "`client.py:40` calls `send()` with no timeout, unlike the
> other three call sites", relevance "you said you are deciding whether the hang
> is in this path".

`incidental` never changes `outcome` and never gates done — which is what stops
the channel from turning into a second, unaccountable report. It is capped at
three for the same reason.

## Done definition

Done is not "found the answer." A scout that swept exhaustively and found
nothing has succeeded; the absence is the finding. Done is: **the question is
settled or provably unsettleable within this brief, and the reach of the search
is on record.**

The scout self-evaluates against the brief it holds. All five checks must pass
before it returns.

| # | Check | Derived from | Failure means |
| --- | --- | --- | --- |
| 1 | Every required brief field is present, and `question` has exactly one reading that would drive a search | `brief.schema.json` required list; `question` | `outcome: blocked` with the matching `blocked_on.reason` |
| 2 | The stopping rule for `breadth` is met, or an affirmative early exit occurred, or `budget` ran out | `breadth`, `territory`, `budget` | `outcome: partial`, remainder in `coverage.not_searched` |
| 3 | Each part of `question` and every `sub_questions` entry is settled affirmatively (file evidence), settled by absence (search evidence), or listed in `unknowns` with `would_settle` | `question`, `sub_questions` | `outcome: partial` |
| 4 | The report validates against `report.schema.json`, and the unencodable cross-field rules hold | both schemas | Fix before returning — an invalid report is not a result |
| 5 | `answer` answers `question` in the form `answer_shape` asked for, and every statement in it traces to a finding | `question`, `answer_shape` | Fix before returning |

Two failure modes this rules out on purpose. **Finding the answer without
recording coverage is not done** — the dispatcher cannot weigh an answer whose
reach is unknown. **An unfinished sweep that found nothing is not
`exhausted_absent`** — it is `partial`, and reporting it otherwise converts a
gap into a false fact, which is the one error in this system that nobody
downstream can detect.

## Failure and edge cases

| Situation | Handling |
| --- | --- |
| A required brief field is missing | `blocked`, `reason: missing_required_field`, `missing_fields` naming them. Do not infer, do not proceed on a plausible default. |
| `question` admits two readings implying different searches | `blocked`, `reason: ambiguous_question`, both readings in `detail`. If the readings imply the *same* search, proceed and record `interpretation`. |
| A path in `territory` does not exist at `repo.revision` | `blocked`, `reason: territory_not_found`. Name the path and any near miss as fact; do not substitute the file you think was meant. |
| The brief asks the scout to choose, rank, or recommend | `blocked`, `reason: decision_requested`. Propose the fact-shaped rewording in `detail`. |
| `budget` exhausted mid-sweep | Stop. `partial`, `stop_reason: budget_exhausted`, remainder in `not_searched` in the order a resumption should take it. |
| The answer requires running code and `may_run` is empty | `partial`; `unknowns[].would_settle` names the exact command. |
| Evidence contradicts `given` or `already_ruled_out` | Report it as a finding with `contradicts_brief: true`, first in the list. Do not silently re-plan the search around a premise you have just disproved — the dispatcher owns its own premises. |
| A better question becomes obvious | `unknowns`, or `incidental` if it bears on `decision_context`. Do not chase it: the budget belongs to the question that was asked. |
| The answer lies outside the repository (dependency internals, runtime, network) | Report the boundary as an absence finding with search evidence, `exhausted_absent` scoped to the repo, and name where the answer would live in `unknowns`. |
| A sibling's territory overlaps yours | Report your own findings only. If you skipped ground on that basis, record it in `not_searched` with the sibling's `dispatch_id` as the reason. |
| The report would exceed a cap | Compress — merge findings, cite ranges instead of quoting. Never drop a `coverage` field to fit, and never truncate silently. |
| The repository is dirty | Report it (`repo.dirty: true`) and proceed. A finding from a dirty tree may not be reproducible from the SHA, and the dispatcher needs to know before acting. |

## Worked example

A real question about this repository at `cba09e3`, exercising the hardest
outcome: an exhaustive sweep that found nothing, where the absence *is* the
answer. Both documents below validate against the schemas in this directory.

### Brief

```json
{
  "version": "scout-brief/1",
  "dispatch_id": "S1-brief-schema",
  "dispatched_by": "odin",
  "issued_at": "2026-08-06T14:02:00Z",
  "repo": {
    "path": "/home/user/valhalla",
    "branch": "claude/agent-roles-scaffold-836v5f",
    "revision": "cba09e3"
  },
  "question": "Does this repository define anywhere a field-level schema for the dispatch brief that an orchestrator sends to a sub-agent role?",
  "decision_context": [
    "I am about to have Mimir plan a shared brief/report envelope covering all five roles.",
    "If a schema already exists I need to extend it rather than invent a competing one; if none exists the plan starts from scratch.",
    "Anything that already constrains the shape of messages between roles counts, including prose."
  ],
  "answer_shape": "yes/no, plus the paths of the closest existing thing if the answer is no",
  "breadth": "exhaustive",
  "budget": {
    "tool_calls": 40,
    "files_read": 60
  },
  "given": [
    "As far as I know the repo is documentation-only and the five roles live under agents/.",
    "Each role folder has a README.md describing the myth, the responsibilities, and the boundaries."
  ],
  "already_ruled_out": [],
  "vocabulary": [
    { "term": "brief", "means": "the message a dispatcher sends to a sub-agent" },
    { "term": "role", "means": "one of the five agents named in agents/README.md" },
    { "term": "the council", "means": "the five roles collectively, per the heading of agents/README.md" }
  ],
  "may_run": []
}
```

### Report

```json
{
  "version": "scout-report/1",
  "dispatch_id": "S1-brief-schema",
  "scout_id": "huginn",
  "completed_at": "2026-08-06T14:11:00Z",
  "repo": {
    "branch": "claude/agent-roles-scaffold-836v5f",
    "revision": "cba09e3",
    "dirty": false
  },
  "breadth": "exhaustive",
  "outcome": "exhausted_absent",
  "answer": [
    "No. Nothing in this repository defines fields, a schema, or any structured message format between roles.",
    "The closest existing thing is a one-line `**Inputs:**` / `**Outputs:**` prose pair in each of the five role READMEs, which names the kinds of things a role takes and returns but gives no field names, no types, and no required/optional distinction.",
    "There is no code, no config, and no template of any kind to extend."
  ],
  "findings": [
    {
      "id": "F1",
      "claim": "No file in the repository defines a schema, contract, field list, envelope, or structured message format for messages between roles.",
      "claim_kind": "absence",
      "evidence": [
        { "kind": "search", "query": "grep -ril 'schema|contract|field|envelope|payload'", "scope": "all 7 tracked files", "hits": 0 },
        { "kind": "search", "query": "grep -ril 'json|yaml'", "scope": "all 7 tracked files", "hits": 0 },
        { "kind": "search", "query": "git ls-files | grep -v '\\.md$'", "scope": "repository root", "hits": 0, "note": "no non-Markdown file is tracked at any path" }
      ],
      "confidence": "high"
    },
    {
      "id": "F2",
      "claim": "The repository is documentation only: 7 tracked files, all Markdown, one root README plus an index and five role READMEs under agents/.",
      "claim_kind": "presence",
      "evidence": [
        { "kind": "file", "path": "README.md", "line_start": 10, "line_end": 16, "quote": "the role/character/folder table listing all five roles" },
        { "kind": "search", "query": "git ls-files", "scope": "repository root", "hits": 7 },
        { "kind": "search", "query": "ls -a", "scope": "repository root", "hits": 3, "note": "README.md, agents/, .git - no hidden config directory" }
      ],
      "confidence": "high"
    },
    {
      "id": "F3",
      "claim": "Each of the five role READMEs states its inputs and outputs as one line of prose rather than as named fields, which is the only existing constraint on inter-role message shape.",
      "claim_kind": "presence",
      "evidence": [
        { "kind": "file", "path": "agents/odin/README.md", "line_start": 36, "line_end": 37 },
        { "kind": "file", "path": "agents/huginn-and-muninn/README.md", "line_start": 36, "line_end": 39, "quote": "**Inputs:** a specific question, and a stated breadth" },
        { "kind": "file", "path": "agents/heimdall/README.md", "line_start": 40, "line_end": 42 },
        { "kind": "search", "query": "grep -rn '^\\*\\*Inputs:\\*\\*|^\\*\\*Outputs:\\*\\*'", "scope": "all tracked *.md", "hits": 10, "note": "two per role README, five roles" }
      ],
      "confidence": "high"
    },
    {
      "id": "F4",
      "claim": "The word 'dispatch' appears three times, all in prose describing that scouts are sent; no occurrence describes what is sent with them.",
      "claim_kind": "presence",
      "evidence": [
        { "kind": "search", "query": "grep -rn dispatch --include=*.md .", "scope": "all tracked *.md", "hits": 3, "note": "agents/README.md:43, agents/huginn-and-muninn/README.md:21 and :49" },
        { "kind": "file", "path": "agents/huginn-and-muninn/README.md", "line_start": 21 }
      ],
      "confidence": "high"
    },
    {
      "id": "F5",
      "claim": "No such schema was present earlier and removed: the entire history is three commits on two branches and no file has ever been deleted.",
      "claim_kind": "absence",
      "evidence": [
        { "kind": "search", "query": "git log --all --oneline", "scope": "all refs", "hits": 3 },
        { "kind": "search", "query": "git log --all --diff-filter=D --name-only", "scope": "all refs", "hits": 0 }
      ],
      "confidence": "high"
    }
  ],
  "coverage": {
    "strategies": [
      { "method": "directory_listing", "detail": "git ls-files for the full tracked set, plus ls -a on the root for untracked or hidden config" },
      { "method": "read_full", "detail": "opened all 7 Markdown files end to end; the whole repository is 300 lines" },
      { "method": "grep_identifier", "detail": "case-insensitive grep over all tracked content for schema, contract, field, envelope, payload, json, yaml, brief, dispatch" },
      { "method": "grep_string", "detail": "structural grep for the existing convention: ^\\*\\*Inputs:\\*\\*|^\\*\\*Outputs:\\*\\*" },
      { "method": "git_history", "detail": "git log --all, plus a deleted-file filter, across main and claude/agent-roles-scaffold-836v5f" }
    ],
    "searched": [
      "README.md",
      "agents/README.md",
      "agents/*/README.md",
      "repository root including hidden entries",
      "full commit history on all local and remote refs"
    ],
    "not_searched": [
      { "area": ".git/", "reason": "internals beyond the commit log; no repository source lives there" },
      { "area": "remote refs other than origin/main and origin/claude/agent-roles-scaffold-836v5f", "reason": "those are the only two remote refs that exist" }
    ],
    "stopped_early": false,
    "stop_reason": "stopping_rule_met"
  },
  "unknowns": [
    {
      "question": "Does such a schema exist outside this repository, in a prior session's notes or a sibling repo?",
      "would_settle": "Naming the other location in a follow-up brief; nothing in this tree points to one and this search cannot see past the repo."
    }
  ],
  "incidental": [
    {
      "observation": "The five Inputs/Outputs lines are not parallel to each other: three name artifacts such as 'the diff, the plan', two name abstractions such as 'prior session context'.",
      "evidence": [
        { "kind": "file", "path": "agents/heimdall/README.md", "line_start": 40, "quote": "**Inputs:** the diff, the plan, the acceptance criteria, the codebase." },
        { "kind": "file", "path": "agents/odin/README.md", "line_start": 36, "quote": "**Inputs:** the user's request, repository state, prior session context." }
      ],
      "relevance": "They are the surface a shared envelope would have to line up with, and you said the plan is a shared envelope."
    },
    {
      "observation": "agents/README.md line 34 says scouts return 'facts, to whoever asked' - the only statement anywhere in the repo about where a sub-agent's output is routed.",
      "evidence": [
        { "kind": "file", "path": "agents/README.md", "line_start": 34, "quote": "scouts - read only ... facts, to whoever asked" }
      ],
      "relevance": "Routing is a field the envelope will need, and this is the only existing statement of intent about it."
    }
  ],
  "effort": {
    "tool_calls": 12,
    "files_read": 7
  }
}
```

Note what the report does not contain: the contents of the seven files. Odin can
read those. What Odin could not do without the raven is know that the sweep came
up empty, how far it reached, and where it stopped.

Reading it in a terminal:

```
jq -r '.outcome, "", .answer[]' report.json
jq -r '.findings[] | "[\(.confidence)] \(.claim)"' report.json
jq -r '.coverage.not_searched[] | "GAP \(.area) — \(.reason)"' report.json
jq -r '.findings[].evidence[] | select(.kind=="search" and .hits==0) | .query' report.json
```

## Notes for the shared contract

Four more role specs are coming. These are the seams where a common envelope
will likely be extracted — flagged, not built.

| Almost certainly shared | Why |
| --- | --- |
| `version`, `dispatch_id`, `dispatched_by`, `repo` (both shapes, including the `dirty` echo) | Pure addressing and tree-pinning. Nothing about them is scout-specific, and every role needs its output pinned to a revision. |
| `decision_context`, `given`, `already_ruled_out`, `vocabulary` | These exist because sub-agents start cold. Every role starts cold. |
| `budget`, `may_run`, `effort` | Effort ceilings and execution grants apply to any delegated worker; only the defaults differ. |
| `$defs.evidence`, `file_evidence`, `search_evidence` | Heimdall already anchors findings to file and line. One evidence vocabulary across roles is what makes findings composable and dedupable — extract this first. |
| `confidence`, `contradicts_brief` | Any role can be handed a false premise and needs a non-silent way to say so. |
| `blocked` and `partial` as outcome values, `blocked_on` with `missing_fields`, and the rule "a missing required field blocks, never guesses" | Underspecification is the shared failure mode. These should be verbatim across all five. |
| The `incidental` channel | The implementer README already describes exactly this behavior — surface the adjacent change, do not slip it in. Same mechanism, different role. |
| The prose-in-JSON convention: single-line strings, arrays for paragraphs, caps | Nothing about it is scout-specific, and inconsistency here would be immediately annoying at the terminal. |
| The five-check done shape (brief valid → work complete per stated stopping rule → each asked part settled or listed → report valid → answer usable) | The skeleton generalizes; only what fills each slot changes. |

| Looks scout-specific | Why, and the probable analogue |
| --- | --- |
| `breadth` and its stopping rules | The *slot* generalizes — every role needs a stated scope with a stopping rule — but the values do not. Heimdall's is diff scope, the implementer's is the plan's steps, Mímir's is depth of investigation. Expect a shared `scope` slot with role-specific enums. |
| `answered` / `exhausted_absent` | The scout's specialization of "settled". Heimdall's are pass/fail, the implementer's are built/blocked. Likely one shared enum with a role-specific tail rather than four disjoint sets. |
| `coverage` (`strategies`, `searched`, `not_searched`, `stopped_early`) | Strongest candidate for surprising generality. Heimdall wants "what of the diff I did not read" just as much; the implementer's analogue is steps-completed versus steps-skipped. Worth designing as shared, with a role-specific `strategies.method` enum. |
| `claim_kind` and the presence/absence evidence rule | Sharpest where absence is a legitimate result, which is mostly reconnaissance. Heimdall may want it for "no test covers this". |
| The indicative-mood rule | Scout-only, deliberately. Mímir and Heimdall are *supposed* to be normative; banning "should" across all roles would break them. |
| `question` / `sub_questions` / `answer` / `answer_shape` | The scout's payload. Each role has its own — plan, diff, verdict. |
| The numeric caps (7 findings, 240-character claims) | Tuned to reconnaissance. The principle — conclusions, not transcripts — generalizes; the values should be re-derived per role. |

Two seams worth deciding before the next spec is written:

1. **Is `outcome` one shared enum with role-specific members, or a per-role enum
   sharing `blocked` and `partial`?** The orchestrator branches on this, and
   that branch will be the busiest logic in the system.
2. **Do the shared parts live in a third schema that the role schemas `$ref`, or
   are they copied into each?** These two files deliberately use no external
   `$ref`, so that a cold agent handed one file has the entire contract. A
   shared envelope trades that property for non-duplication, and the trade
   should be made once, on purpose.
