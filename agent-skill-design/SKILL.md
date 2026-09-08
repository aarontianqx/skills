---
name: agent-skill-design
description: Design or review skills for AI agents. Use when creating, restructuring, or auditing a skill's trigger description, domain guidance, workflow, references, scripts, tools, or optional companion CLI. Covers deterministic agent behavior, scoped instructions, safe boundaries, and stable interfaces for automation.
---

# Design an agent skill

Treat a skill as a decision guide for an agent. Its job is to supply the domain knowledge, boundaries, and workflow decisions that the agent cannot reliably infer from the user's request or from a tool's raw output.

A skill does not need a CLI. When a task benefits from reliable API access, querying, batch operations, or state changes, a thin CLI can be a useful companion. Design the skill first and add a CLI only when it provides a real execution surface.

> Make the agent's next action determinable without guessing, parsing unstable prose, recalling hidden knowledge, or taking an unreviewed consequential action.

## 1. Define the skill's job

Start with the user scenarios the skill should handle:

- what the skill does and what it does not do;
- which requests should trigger it, including requests that do not name a tool;
- the domain model or mental model the agent needs;
- the boundary between this skill and nearby skills or general reasoning.

Do not introduce a CLI, reference file, or fixed workflow merely because it is common in another skill. Add a resource when it changes decisions or makes execution materially more reliable.

## 2. Write the trigger description

The frontmatter description is the routing contract and is available before the body is loaded. Make it concise and discriminating:

- describe the capability and the situations in which users need it;
- include important implicit triggers, not only tool names or command verbs;
- mention a boundary when it prevents likely misrouting;
- avoid exhaustive operation lists, implementation history, and generic claims about being agent-friendly.

Use the body for detailed workflows and decision rules. Do not put volatile schemas, long command references, or design rationale in the description.

## 3. Keep only non-inferable knowledge

Include facts that change the agent's decisions:

- state transitions and independent state axes;
- surprising side effects or irreversible operations;
- lossy fields or UI projections that must not drive decisions;
- defaults, permissions, and conditions for asking, stopping, or branching;
- what evidence is sufficient to say the task is complete.

Remove generic advice, obvious mechanics, and duplicated API documentation. Keep each fact in one authoritative place. Put large or conditionally relevant material in focused references and say exactly when to read each one.

Keep implementation details in their owning layer. A skill normally should not describe token parsing, HTTP construction, parser internals, or response-shaping code unless the agent must make a decision based on that behavior.

## 4. Choose the execution surface

Use the lightest surface that makes the requested work reliable:

- a skill without tools is appropriate for reasoning, writing, review, and domain workflows;
- an existing application or API may be enough when the agent can use it safely and readably;
- a CLI is appropriate when repeatable queries, structured mutations, batching, or a stable local interface materially improve reliability;
- scripts are appropriate for deterministic transformations that would otherwise be reimplemented in conversation.

When a CLI is added, keep it thin: parse inputs, call the existing API, and expose the facts and operations the agent needs. Keep authorization, state machines, and domain invariants in the server or source system. Fix shared contract defects at their source instead of adding CLI-specific guesses.

## 5. Define the interface contract when tools are present

Optimize for deterministic branching, not for JSON as an end in itself.

- Keep successful output stable and free of unrelated progress or diagnostics.
- Use structured output for domain objects, collections, and results that agents must inspect or combine.
- Stable, field-oriented text is acceptable when the primary result is inherently textual, such as logs, diffs, or diagnostics.
- Make the default format explicit. If alternate formats exist, define their intended consumers and semantics.
- Make errors branchable with stable categories, codes, and exit statuses. Keep human messages separate from actionable fields; never require parsing message prose.
- Expose empty results, truncation, pagination, partial success, and retry information explicitly.
- Provide local introspection when the command surface is large or volatile, such as help or schema commands.

An interface contract includes setup behavior. Do not assume credentials are already configured:

- distinguish a missing configuration file, a missing field, invalid credentials, insufficient permissions, and network failure;
- tell the agent what the user must provide or change when setup is missing;
- if tools share a configuration file, update only owned fields and preserve unrelated fields;
- never print credentials in output, diagnostics, or examples.

## 6. Design safe operations

For read-only work, define how the agent should interpret empty, partial, stale, or ambiguous results and what evidence supports the answer.

For writes, expose the smallest useful operation set and make consequential actions recoverable where possible:

- prefer reversible or idempotent operations;
- provide a preview, dry-run, confirmation, or equivalent risk control appropriate to the operation;
- make scope and target explicit before mutation;
- re-read the resulting state when command success alone does not prove the business outcome;
- expose per-item outcomes for batches so failed items can be retried safely.

Do not impose write-specific ceremony on a purely read-only skill. Match the process to the actual risk.

## 7. Keep the skill and tools aligned

Verify current implementations before documenting commands, fields, permissions, or semantics. Treat proposals and old documentation as intent, not proof.

When a skill and a CLI are distributed together, keep their contracts version-compatible and make drift detectable. Embedding or dynamically exposing the canonical skill from the CLI can help, but it is an option rather than a universal requirement.

When the same fact is owned by an API, CLI, skill, or reference, avoid copying it into the other layers. The skill should explain how to decide; the tool should reliably execute and report; the source system should enforce the business truth.

## 8. Validate from evidence

Start from a realistic user request, failure, trace, or output artifact. Locate each problem in the correct layer: source system, API, tool, skill workflow, reference, or deterministic script. Make the smallest root-level change and remove guidance that the change makes obsolete.

Validate the final artifact at the level its risk requires:

- inspect frontmatter, links, command help, and referenced resources;
- run structural validation and targeted regression checks;
- test representative raw inputs and outputs for tools;
- use an independent fresh-agent pass for complex or consequential skills when it adds meaningful confidence.

Before considering the design complete, verify that an agent with no conversation history can determine the default action, every consequential branch, when to ask or stop, and how to prove the requested outcome.
