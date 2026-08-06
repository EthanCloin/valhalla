# The Council

Five roles, each with a folder here. Each folder's README covers the myth, the
role's responsibilities and boundaries, and why the two were paired.

| Role | Character | Folder |
| --- | --- | --- |
| Orchestrator | Odin | [`odin/`](odin/) |
| Scout | Huginn and Muninn | [`huginn-and-muninn/`](huginn-and-muninn/) |
| Planner | Mímir | [`mimir/`](mimir/) |
| Implementer | The Sons of Ivaldi | [`sons-of-ivaldi/`](sons-of-ivaldi/) |
| Reviewer | Heimdall | [`heimdall/`](heimdall/) |

## The flow

```
                    ┌──────────────┐
     request ──────▶│     ODIN     │──────▶ report
                    │ orchestrator │
                    └───┬───▲──┬───┘
              delegates │   │  │ findings
                        ▼   │  ▼
   ┌────────────┐   ┌───────┴──────┐   ┌──────────────┐
   │   MÍMIR    │──▶│ SONS OF      │──▶│   HEIMDALL   │
   │  planner   │   │ IVALDI       │   │   reviewer   │
   │            │   │ implementer  │   │              │
   └─────┬──────┘   └──────┬───────┘   └──────┬───────┘
         │                 │                  │
         └───────┐         │        ┌─────────┘
        questions │        │        │   any role may send scouts
                  ▼        ▼        ▼
             ┌───────────────────────────┐
             │     HUGINN & MUNINN       │
             │    scouts — read only     │──▶ facts, to whoever asked
             └───────────────────────────┘
```

Odin holds the context and decides what runs. Mímir plans without writing.
The Sons of Ivaldi build to the plan without judging it. Heimdall judges
without fixing. Findings return to Odin, who decides whether the loop runs
again.

The ravens sit outside the pipeline rather than in it. They are dispatched —
often several at once — whenever a role needs to know something about the
codebase that nobody has established yet, and they answer to the role that
sent them.

## The separations that matter

The value of splitting these roles comes entirely from what each one is *not*
allowed to do:

- **Planner ≠ implementer.** A role that has already started building has sunk
  cost in its approach and will defend it.
- **Implementer ≠ reviewer.** Nobody finds their own blind spots by looking
  harder at the same code.
- **Reviewer ≠ fixer.** A reviewer that patches what it finds ends up reviewing
  its own work on the next pass.
- **Scout ≠ decider.** The ravens report what is there. The moment a scout
  starts recommending, its findings stop being neutral input.
- **Orchestrator ≠ any of them.** Odin's judgment of a review is only worth
  something if Odin didn't write the code.

Not every request needs the full council. A one-line fix routed through all
five is worse than the fix. Odin decides.
