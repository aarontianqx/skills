---
name: agent-cli-design
description: Design or review an agent-first CLI and its companion skill as a thin, machine-readable client over an existing system API. Use when deciding the CLI's I/O and error contract, command and safety surface, API boundaries, domain model exposure, or the skill that teaches agents to operate it.
---

# Design an Agent-First CLI

Treat the CLI and its companion skill as one product: a contract an agent can branch on reliably.

> Design every surface so the agent's next action is determinable without guessing, parsing prose, recalling hidden knowledge, or firing a request blind.

## 1. Make the CLI a machine contract

- **Keep stdout machine-readable.** Emit exactly one structured success value to stdout. Send progress, warnings, notices, and diagnostics to stderr.
- **Make errors branchable.** Return a stable category and code, a human message, and separate actionable fields such as the missing role or retry target. Map categories consistently to exit codes. Never require the agent to parse message text. Include the literal next command or remedy when it is unambiguous.
- **Support on-demand introspection.** Let the agent inspect a command's inputs, permissions, and risk without carrying the whole API surface in context. Keep introspection of the client contract unauthenticated, side-effect free, and locally available. A `schema <command>` verb is one suitable design.
- **Make safety guarantees exact.** Let agents preview writes or provide an equivalent risk-control mechanism. If `--dry-run` promises a local preview, perform full local validation, print the exact request, and do not touch the network. Never print secrets.
- **Represent collection state honestly.** Expose pagination metadata and a way to continue or fetch all results. Return per-item batch outcomes so partial failures are directly retryable.

## 2. Put behavior and truth in the right layer

- **Verify current code before designing against it.** Treat proposals and docs as intent; confirm the actual API, fields, and semantics in the implementation.
- **Reuse before extending.** Exhaust reasonable compositions of existing read, query, and batch interfaces before adding an endpoint. Add one only when the existing surface cannot reach the goal with acceptable correctness or round trips.
- **Keep the CLI thin.** Put state machines, authorization, and domain invariants on the server. Do not duplicate authorization checks in the CLI; surface the server's failure and exact required role. Keep the normal path close to parse inputs → call the API → render the result. Fix shared contract defects at the source instead of teaching CLI-specific workarounds.
- **Expose facts, not lossy UI projections.** Return the domain's independent state axes. Do not make agents branch on a fused display status that can misrepresent edge states.
- **Make writes recoverable.** Prefer reversible operations and gate or omit unnecessary irreversible ones. Make writes safely repeatable; prefer server-side idempotency, and keep any unavoidable CLI-side resolution explicit and minimal.
- **Detect contract drift early.** Version the client against the server contract so incompatible changes fail as early as practical, ideally at build time.

## 3. Write the companion skill as a decision guide

Assume the agent reading the skill is already installed and configured. Teach it how to reason about the domain and use the CLI, not how the skill itself was built.

- **Trigger on user scenarios.** In frontmatter, state what the skill does and the situations in which users need it, including requests that do not name the tool. Avoid verb catalogs, over-broad or alarming triggers, and design-history narration.
- **Lead with the domain and the work.** Establish the minimum mental model before the workflow. Put setup and authentication guidance next to the failures that make it actionable, not in a mandatory preamble.
- **Spend context only on non-inferable knowledge.** Include surprising side effects, lossy fields, domain constraints, and decision boundaries. Cut generic advice and obvious mechanics; state domain-specific defaults whenever they affect branching or user-visible outcomes.
- **Match precision to the knowledge.** State invariants exactly. Express variable decisions as a safe default plus precise conditions for asking, stopping, or branching. Resolve volatile schemas, commands, and component details through current introspection or references rather than copying them into the workflow.
- **Give each fact one owner.** Keep workflow and selection guidance in the main skill; keep independently owned or conditionally relevant details in direct references. Split by ownership, change cadence, or conditional relevance—not by arbitrary file count—and state when each reference must be read.
- **Define completion, not just actions.** Tell the agent how to re-read and verify the resulting system state or artifact. A successful command or syntax validator alone is not proof of business correctness.
- **Fix causes instead of adding warnings.** Remove instructions that induce unwanted behavior. If the CLI or API cannot expose a required fact reliably, repair that contract rather than compensate with prose. Keep warnings only for states proven reachable in the current system.
- **Version the skill with the CLI.** Make drift detectable. Embedding the canonical skill in the binary and exposing it through a read command is one strong implementation, not the only one.

## 4. Iterate from evidence

1. Start from a real user request, failure, trace, or output artifact.
2. Locate the missing knowledge or defect in the correct layer: server, API, CLI, skill workflow, reference, or deterministic script.
3. Make the smallest root-level change and remove guidance it makes obsolete.
4. Run structural validation and targeted regression tests.
5. Forward-test substantial changes with a fresh agent, the real skill, and representative raw inputs. Do not leak the expected answer, suspected bug, or prior conclusions. Use a sandbox or draft, and obtain approval before a test would mutate live state.
6. Inspect the final artifact and remote state, not only command success. If forward-testing succeeds only with conversation history or leaked context, the skill is incomplete.

Before considering the design complete, verify that a fresh agent can determine the default action, every consequential branch, when to ask or stop, and how to prove the requested outcome.
