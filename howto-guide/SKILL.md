---
name: howto-guide
description: Write or revise operational how-to documentation (操作手册、使用指南、教程、SOP、runbook、FAQ) for internal tools, admin consoles, and business processes. Use when the user asks for a task-oriented guide that walks readers through doing something in a UI or a process, with or without screenshots. Not for architecture docs, ADRs, design proposals, or code-level reference docs.
---

# Write an operational how-to guide

## Scope

Handle requests for 操作手册 / 使用指南 / 教程 / SOP / runbook / FAQ / 操作说明 / 流程文档 — documents whose job is to let a reader **complete a task in a UI or a business process**. The reader may be ops, PM, support, or a new teammate.

Do **not** use this skill for architecture overviews, design proposals/ADRs, code comments, API references, or debugging investigations: those follow the owning repo's own conventions, not this skill.

## Workflow

1. **Verify before writing.** Read the real UI or process when it is reachable, using whatever browser or inspection tooling is available. Labels, entry points, and defaults change; the guide must match the deployed UI, not an older mental model. Walk through on a test environment when possible, and never mutate production just to produce material.
2. **Draft with the rules below.** Keep only content the reader cannot infer by looking at the screen.
3. **Acceptance pass.** A reader with zero context can finish the task by following the steps, every label matches the live UI, and no sentence exists only to reassure or to address the reader directly.

## Structure

- **Task-ordered, not module-ordered.** Follow the real job sequence (e.g. create → configure → test → ship → export), not the system's module tree.
- **Action headings.** Each section is one step the reader performs (创建问卷 / 圈选目标人群 / 配置弹窗), not a topic (问卷系统简介).
- **Every step answers three things**: where to go, what to do, what value to enter. If a step has none of these, it is not a step — cut it.
- **Environment table first when multiple envs exist**: test vs prod URLs (console, user-facing page) in a small table at the top, so nobody pastes test data into prod by mistake.
- **End with a short FAQ** of the 3–6 failures that actually happen (blank page, permission denied, not visible, quota full). Each answer is 1–2 lines with the concrete cause and the check or fix.

## Text: what to keep and what to cut

Keep only these three kinds of sentences:

1. **Values and formats**: group naming rules, URL templates, CSV format, priority suggestions, required vs optional, and what can no longer be edited later.
2. **Consequences and reversibility**: when it takes effect, whether it can be changed or undone, what "full/offline/expired" looks like to a user.
3. **Order dependencies**: what must be done before another step (e.g. clear test submissions before going live).

Cut these without mercy:

- Self-addressing and meta talk: 「本手册面向…」「这是给你看的」「SOP」 as a title suffix, revision history, "why I wrote this".
- Reassurance filler: 「不用紧张」「就行」「更方便」「很简单」.
- UI paraphrase: do not narrate what the screenshot already shows.
- Purpose-of-the-doc throat clearing: 「比较稳妥的做法是…」「推荐流程是…」 — just give the steps.
- Scaffold: bold spans everywhere, ①②③, 「三个要点」, 「一句话总结」.

## Tone

Write like a normal internal process doc, in the reader's language:

- Imperative, present tense, concrete verbs (点击、填写、选择、导出), short declarative sentences.
- Professional but not stiff: no corporate boilerplate, no AI-flavored parallelisms, no chatty colloquialisms (哦、呢、嘛、就好了).
- Use the exact labels from the live UI, in 「」quotes; bold only the 2–3 things that genuinely need emphasis per page.

## Images and text ratio

A screenshot earns its place only when the reader must **locate a control or a field visually**: editors, complex forms, upload dialogs, modal actions, filter bars.

Do not include: navigation lists, homepages, tables of contents, screenshots that merely repeat the previous one, or any image that adds no new information.

Rules of thumb:

- One image per non-trivial step, placed **immediately after** the step it illustrates; ~6–8 images for a 6-step guide is typical.
- Crop to the working area; no full-screen chrome unless the layout itself is the point.
- Never capture real user IDs, submission contents, staff emails, tokens, or prod data. Use test fixtures and the test env, and view each image before including it.

## Publishing

- Follow the target site's own conventions for navigation, rendering checks, and deployment; do not assume any particular site or that other files (including AGENTS.md) need editing.
- Verify the published page once: images load, links resolve, no raw markdown artifacts (literal `**`, broken tables).
- Report the URL and a 2–3 line summary of what changed.
