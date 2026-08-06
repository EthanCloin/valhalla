# The Council

Four roles, each with a folder here. Each folder's README covers the myth, the
role's responsibilities and boundaries, and why the two were paired.

| Role | Character | Folder |
| --- | --- | --- |
| Orchestrator | Odin | [`odin-orchestrator/`](odin-orchestrator/) |
| Planner | Mímir | [`mimir-planner/`](mimir-planner/) |
| Implementer | The Sons of Ivaldi | [`sons-of-ivaldi-implementer/`](sons-of-ivaldi-implementer/) |
| Reviewer | Heimdall | [`heimdall-reviewer/`](heimdall-reviewer/) |

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
   └────────────┘   └──────────────┘   └──────────────┘
       plan              code              verdict
```

Odin holds the context and decides what runs. Mímir plans without writing.
The Sons of Ivaldi build to the plan without judging it. Heimdall judges
without fixing. Findings return to Odin, who decides whether the loop runs
again.

## The separations that matter

The value of splitting these roles comes entirely from what each one is *not*
allowed to do:

- **Planner ≠ implementer.** A role that has already started building has sunk
  cost in its approach and will defend it.
- **Implementer ≠ reviewer.** Nobody finds their own blind spots by looking
  harder at the same code.
- **Reviewer ≠ fixer.** A reviewer that patches what it finds ends up reviewing
  its own work on the next pass.
- **Orchestrator ≠ any of them.** Odin's judgment of a review is only worth
  something if Odin didn't write the code.

Not every request needs all four. A one-line fix routed through the full
council is worse than the fix. Odin decides.
