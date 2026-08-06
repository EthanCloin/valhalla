# valhalla

A set of agent roles covering a development workflow — orchestration,
planning, implementation, and review — themed on Norse mythology.

Each role lives in [`agents/`](agents/) with a README describing the myth, the
role's responsibilities and boundaries, and how the two connect.

| Role | Character | |
| --- | --- | --- |
| Orchestrator | Odin | [`agents/odin-orchestrator/`](agents/odin-orchestrator/) |
| Planner | Mímir | [`agents/mimir-planner/`](agents/mimir-planner/) |
| Implementer | The Sons of Ivaldi | [`agents/sons-of-ivaldi-implementer/`](agents/sons-of-ivaldi-implementer/) |
| Reviewer | Heimdall | [`agents/heimdall-reviewer/`](agents/heimdall-reviewer/) |

See [`agents/README.md`](agents/README.md) for how the roles hand off to each
other and why the boundaries between them are drawn where they are.
