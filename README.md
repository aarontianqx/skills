# skills

A personal collection of reusable [Agent Skills](https://docs.claude.com/en/docs/claude-code/skills) — focused, model-invoked playbooks that teach a coding agent how to do a specific kind of work well.

## Layout

One directory per skill, each holding a `SKILL.md`:

```
<skill-name>/
└── SKILL.md   # YAML frontmatter (name, description) + Markdown instructions
```

A skill may also carry `references/` (docs loaded on demand), `scripts/`, or `assets/` when it grows beyond a single file. The frontmatter `description` is the trigger — it states what the skill does and the situations in which to use it.

## Skills

| Skill | What it's for |
|-------|---------------|
| [`agent-skill-design`](agent-skill-design/SKILL.md) | Designing skills for AI agents, including domain guidance, workflows, references, tools, and optional companion CLIs with stable agent-facing contracts. |
| [`imagegen`](imagegen/SKILL.md) | Generating or editing raster images via the OpenAI Image API (official or any OpenAI-compatible relay/proxy endpoint) — a vendor-neutral port of the Codex `imagegen` skill with explicit endpoint/key configuration. |

## Installing a skill

Skills are discovered from a skills directory. To use one from this repo, place it where your agent looks — symlinking keeps it in sync with the repo:

```bash
# discoverable by Claude Code (and other agents reading ~/.claude/skills)
ln -s "$PWD/agent-skill-design" ~/.claude/skills/agent-skill-design
```

If you keep a canonical store at `~/.agents/skills`, link there and mirror into `~/.claude/skills` to match how managed skills are laid out.

## License

See [LICENSE](LICENSE).
