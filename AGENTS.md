# AGENTS.md

Canonical agent instructions for this repo. Start with [`README.md`](README.md)
and [`agents/README.md`](agents/README.md) for what this repo contains and how
the roles hand off to each other.

## Agent skills

### Issue tracker

Issues live in the GitHub Issues of `EthanCloin/valhalla`. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical triage roles, using the default label strings unchanged. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context — one `CONTEXT.md` plus `docs/adr/` at the repo root. See `docs/agents/domain.md`.

## Defining a role

Every role in `agents/` is defined by four things. A spec missing any of them is
incomplete.

1. **Responsibility.** What the role is for, and what it may not do — see the
   separations in [`agents/README.md`](agents/README.md).
2. **Input contract.** The shape and schema of the commands the role receives.
3. **Output contract.** The shape of the result it returns, with clear
   delineations between its parts.
4. **Done definition.** How the role knows it has finished, evaluated against
   the input contract it was given.

### Both contracts must stand alone

Agents here do not share a conversation. A sub-agent never sees the exchange
between the orchestrator and the human, and the orchestrator never sees the
sub-agent's reasoning, searches, or dead ends — only its report. Both ends of
every handoff are read cold.

Every role can read this repository, and that is the richer channel. Briefs and
reports are temporary; committed documentation is durable and shared, so
anything worth more than a single handoff belongs there — and a contract can
point at it instead of restating it.
