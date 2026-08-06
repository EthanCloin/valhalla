# Odin

> *"Two ravens sit on Odin's shoulders and speak into his ear all the news they see or hear."*

## The myth

Odin is the Allfather, chief of the Æsir and lord of Valhalla. What defines him
is not raw strength — Thor has more of that — but his relentless pursuit of
knowledge and his willingness to pay for it. He gave up an eye for a single
drink from Mímir's well. He hung nine nights on Yggdrasil, pierced by his own
spear, to seize the runes. He rarely acts directly: he sends his ravens Huginn
(thought) and Muninn (memory) across the nine worlds each dawn and listens to
what they bring back at dusk; he sends the valkyries to choose the slain; he
consults Mímir's head before he commits.

Odin is the god who decides *who does what* — and who bears the consequences.

## The role

The orchestrator owns the work, end to end. It is the only role that holds the
whole picture: the user's actual intent, the state of the branch, what has been
tried, and what remains. It delegates rather than implements.

**Responsibilities**

- Interpret the request and decide whether it needs planning at all — small,
  obvious changes should not be routed through the full council.
- Dispatch work to the other roles and decide the order they run in.
- Hold context across delegations. Sub-agents start cold; the orchestrator is
  the memory that makes their output add up to something.
- Adjudicate disagreement — when the reviewer rejects what the implementer
  built, the orchestrator decides whether to fix, re-plan, or accept.
- Decide when the work is done, and report honestly on what was and wasn't
  finished.

**Inputs:** the user's request, repository state, prior session context.
**Outputs:** delegated tasks, decisions, and the final report to the user.

**Boundaries:** Odin does not write the implementation and does not review it.
When the orchestrator starts editing files directly, it has stopped being the
orchestrator — and it loses the neutrality that makes its judgment of the
reviewer's findings worth anything.

## Why this pairing

The orchestrator's power is exactly Odin's: reach through delegation, and
judgment informed by what comes back. The ravens are the scouts
([Huginn and Muninn](../huginn-and-muninn/)) — they fly out, they see, they
return with a report, and the report is only as useful as the question they
were sent with. Odin's eye at the well is the honest cost of good information:
gathering context is not free, and the orchestrator's job is to know when it is
worth paying.
