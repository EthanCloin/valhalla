# valhalla

A set of agent roles covering a development workflow — orchestration,
reconnaissance, planning, implementation, and review — themed on Norse
mythology.

Each role lives in [`agents/`](agents/) with a README describing the myth, the
role's responsibilities and boundaries, and how the two connect.

| Role | Character | |
| --- | --- | --- |
| Orchestrator | Odin | [`agents/odin/`](agents/odin/) |
| Scout | Huginn and Muninn | [`agents/huginn-and-muninn/`](agents/huginn-and-muninn/) |
| Planner | Mímir | [`agents/mimir/`](agents/mimir/) |
| Implementer | The Sons of Ivaldi | [`agents/sons-of-ivaldi/`](agents/sons-of-ivaldi/) |
| Reviewer | Heimdall | [`agents/heimdall/`](agents/heimdall/) |

See [`agents/README.md`](agents/README.md) for how the roles hand off to each
other and why the boundaries between them are drawn where they are.
