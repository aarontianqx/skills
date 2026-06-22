---
name: agent-cli-design
description: Design a command-line interface (binary + companion agent skill) whose primary user is an AI coding agent rather than a human, as a thin client over an existing system's API. Use when building, extending, or reviewing a CLI meant to be driven by agents — deciding its I/O contract, error model, command surface, what to expose vs hide from the underlying API, whether to add or reuse backend endpoints, and how to write the skill that teaches an agent to use it.
---

# Designing an Agent-First CLI

This skill is for building a CLI whose main consumer is an **AI agent**, not a human, layered as a thin client over an existing system's API. It also covers the companion **skill** that teaches an agent to drive that CLI. The two are one product: a contract a machine can branch on reliably.

The north star, from which almost everything else follows:

> **The reader is an agent.** A human scans, infers, and tolerates prose. An agent parses, branches, and acts. Design every surface so the agent's next action is *determinable without guessing* — never by parsing prose, never by recalling hidden knowledge, never by firing a request blind.

Work the three layers below in order. They are not independent: the I/O contract shapes what the skill can promise, and the "expose vs hide" decisions shape both.

## 1. The I/O contract (the highest-leverage layer)

Get this right first; the rest composes on top.

- **stdout is data; stderr is everything else.** On success, print exactly one structured object (e.g. a JSON envelope) to stdout. Warnings, progress, notices, diagnostics all go to stderr. An agent must be able to consume stdout whole without scraping fields out of a human table.
- **Errors are an API contract, not a message.** The exit code is a pure function of one error category enum (auth / permission / not-found / precondition / validation / network / …). The structured error carries each fact as its own field — a machine-branchable code, the human message, and any actionable datum (e.g. the exact missing role). The agent branches on the *fields*; it must **never parse the prose message**. If the backend collapses distinct failures into one coarse status on the wire, that is a contract defect worth fixing at the source — see §2.
- **Carry the next action in the error.** Where a failure has an obvious remedy, put the literal next command (or the exact missing role) in a structured `hint`/`role` field, so the agent acts without inferring it.
- **Make the tool self-describing.** Provide a `schema <command>` introspection that returns one command's inputs, the role it needs, and its risk — so the agent inspects on demand instead of carrying the whole API surface in its context. Introspection needs no auth and is free.
- **Every write previews with `--dry-run`, and dry-run must truly not touch the network.** It runs the full *local* chain (flag/enum validation, file/stdin resolution, the command's own checks) and prints the exact request it would send, then stops. A dry-run that dials during resolution is a broken promise; the same goes for "the token is never printed." These safety guarantees are part of the contract — hold them exactly.
- **Paginate honestly.** List output must signal whether more pages exist (a returned/total/has-more/next-cursor block) and offer a way to fetch all. An agent must never be led to assume the first page is the whole set.
- **Partial-failure is machine-diffable.** Batch verbs return per-item results so a half-successful batch can be retried for only the failed items.

## 2. Interfaces: extreme restraint before adding anything

The strongest discipline here is **not** "the CLI needs X, so add endpoint X."

- **If an existing interface can do it, do not add one.** Before proposing any new endpoint, *exhaust the combinations of what already exists*. Most "missing" capabilities — derived state, aggregate statistics, multi-axis views — turn out to be composable from current read/query/batch endpoints. Only add an endpoint when you have verified no existing combination reaches the goal in an acceptable number of round-trips.
- **Verify against the code, not against the design doc.** A proposal states intent; the code states fact. Before building on an interface, confirm it actually exists and that its semantics are what they were described to be. Assumptions about field names, flags, and behaviors that "should" be there are a common source of wrong implementations.
- **A "for the CLI" fix that repairs a shared contract belongs at the source.** If the change you need (e.g. emitting a structured error code instead of a stringified one) actually fixes a defect every client of that API shares, make it server-side rather than working around it in the CLI. The CLI then reads the improved contract with no special-casing.
- **Judge conventions against the nearest neighbor, not the whole repo.** When deciding whether to add validation annotations, error styles, naming, etc., the reference frame is the *immediate sibling* messages/endpoints in the same subsystem — not a repo-wide statistic. Introducing a new convention into a subsystem that has consistently avoided it is a subsystem-level decision, not something a small change should start unilaterally.

## 3. Keep the CLI thin, and expose the *essential* model

- **No business logic in the CLI.** The state machine, authorization, and invariants live server-side. The CLI carries the caller's credentials, is bound by the same rules, and does: parse flags → one call → render. It never re-checks roles locally; it surfaces the server's authorization failure (with the exact required role) so the agent learns from the failure.
- **Expose the domain's orthogonal facts, not the operations-UI's lossy projection.** Systems built for human operators often fuse several independent truths into one display status — and such fused fields are frequently *lossy* (they mislabel edge states). For an agent, expose the underlying independent axes and let it reason on those. Any field that can be wrong, even at the margins, must never be something a client branches on — at most keep it as a clearly-labeled display hint.
- **Hide irreversible/dangerous capabilities.** Prefer recoverable verbs (e.g. archive) and do not expose hard-destructive ones. The agent should not be one mistaken call away from unrecoverable loss; gate or omit such operations.
- **Add client-side idempotency where the server lacks it.** If a create call errors on a duplicate key rather than upserting, implement an idempotent verb in the CLI (resolve-then-branch: create / reuse / restore) so an agent can safely re-run.
- **Pin the client to the same API/contract version as the server**, so the typed client tracks the contract and an incompatible change fails fast (at build) instead of silently at runtime.

## 4. The companion skill: a manual for an agent that is already set up

The skill is the agent's playbook. The failures here are subtle and high-signal — most come from forgetting *who* reads it.

- **Open with the domain and the work, not with setup.** The agent reading the skill is already installed and configured. Lead with what the system is and the mental model it must reason in. Push setup/auth down to a *troubleshooting* note placed where it becomes actionable (next to the error rule for auth failures) — not as a mandatory preamble. Never instruct the skill to install itself; by the time it's read, it's installed.
- **Spend words only on what the agent cannot infer.** Non-obvious facts earn their place: the orthogonal state model, a verb that has a surprising side effect (e.g. "approving also publishes to a channel"), a field that is lossy and must not be trusted. Default behavior needs no instruction. Before keeping any sentence, ask: *would a capable agent do this anyway?* If yes, cut it.
- **Fix behavior at the root; don't paper over it with prose.** If the agent does something unwanted (e.g. eagerly verifying identity before every task), find the instruction in the skill that *induces* it and remove that — do not add a sentence telling it not to. "Do not think about X" only plants X. Suppressing a symptom with text while leaving the cause is the most common skill-writing mistake.
- **Trigger on scenarios, not a verb catalog.** The description frontmatter is what decides whether the skill is invoked. Write one line of *what it is* and one line of *when to use it phrased as the situations a user actually describes* — including the cases where the user needs it but won't name the tool. Avoid a long list of every operation, avoid alarming or over-broad words, and never narrate your design/debate process into the artifact: the agent does not care how the skill was made.
- **Every warning must map to a reachable situation.** Before writing "do not trust X" or "beware Y", prove that situation actually occurs in the current implementation — that the field is really emitted, that the state is really producible. A warning for a state that only the old/buggy code could create, or that no real user can reach, is dead guidance: cut it. And if the same fact surfaces under more than one name, the warning must cover all of them.
- **Keep it one file until length forces a split.** Progressive disclosure (splitting into reference files) is justified when the main file genuinely grows past a few hundred lines, or a single command needs a large flag matrix that would blow the context budget — not because some other skill is multi-file. Splitting a short skill just adds indirection (extra reads) for context you aren't short on.
- **Embed the skill in the binary so it can never drift from the commands that exist.** Surface it via a `read` verb; if an externally-installed copy drifts, nudge the user to update. This prevents the skill from teaching a command the binary no longer has.

## The one-line summary

Building an agent-first CLI is designing a **contract a machine can reliably branch on**: every fact the agent can't infer is explicitly encoded, every fact it *can* infer is cut, and nothing requires parsing prose or guessing. Interface, I/O, and skill all serve that single rule.
