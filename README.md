# Attio Agent Builder

A Claude Code skill for building Custom Agent blocks in Attio workflows.

## The problem

Attio's Custom Agent block is powerful but underspecified. The UI gives you five fields -- prompt, output schema, system prompt, model, and tools -- with no guidance on how they interact. So you end up:

- **Guessing at architecture.** Should this be one agent or three? When should you gate with If/Else? When does a code block beat a second agent?
- **Fighting JSON output.** Some models add preamble text that breaks Attio's JSON parser. You discover this after deploying, not before.
- **Missing edge cases.** Your classifier works on the obvious cases and fails on the ambiguous ones -- fintech that builds software, holding companies with portfolio signals, sparse websites.
- **No testing strategy.** You iterate by running the workflow on one record at a time and eyeballing the output. Regressions go unnoticed.

This skill encodes prompt engineering patterns learned from building and iterating on production Attio agents -- structured output ordering, consistency rules, sequential gating, JSON compliance, and edge case handling.

## What this skill does

Two modes:

- **Plan First** (default) -- Asks questions to understand the full workflow, proposes a multi-block architecture with test cases, then builds each agent one at a time.
- **Quick Build** -- Skips planning. Minimal questions, then straight to paste-ready output.

For each Custom Agent block, it produces output matching the Attio UI field order:

1. **Prompt** -- the main instruction, with variables at the bottom
2. **Output Schema** -- sample JSON for Attio's schema editor
3. **System Prompt** -- classification rules, consistency constraints, role definitions
4. **Model** -- recommendation with rationale
5. **Tools** -- web access, CRM tool access, connected apps

You copy each section directly into the Attio workflow editor. No translation step.

## Install

Copy the directory to your Claude Code skills location:

```
# Per-user (available in all projects)
cp -r attio-agent-builder ~/.claude/skills/attio-agent-builder

# Per-project (available only in this repo)
cp -r attio-agent-builder .claude/skills/attio-agent-builder
```

Claude Code will detect the skill automatically.

## Usage

### Plan First (default)

```
> Build an Attio agent that classifies companies by industry
```

The skill will ask questions about your workflow, propose an architecture (how many agents, what each does, where to gate), suggest test cases, and then build each agent block.

### Quick Build

```
> Quick build: agent that extracts company description from their website
```

Skips planning. Asks minimal clarifying questions, then produces paste-ready output.

### Configuration

Set your default mode in `config.md` inside the skill directory:

```markdown
# Attio Agent Builder Config
default-mode: quick
```

Either mode can be overridden per invocation by saying "plan this" or "quick build".

## What's inside

| File | Description |
|------|-------------|
| `SKILL.md` | Skill definition -- modes, output format, key rules, invocation logic |
| `references/architecture.md` | Workflow building blocks, when to split agents, composition patterns |
| `references/prompt-patterns.md` | 14 prompt engineering patterns with failure modes and fixes, each with source attribution |
| `references/tools.md` | Web access, CRM tool access, and connected apps -- what to enable and when |
| `references/examples.md` | Three complete working agents: company research, classification, data validation |
| `references/variables.md` | Attio variable syntax, placement rules, common attributes |
| `references/testing.md` | Test case design, safe testing patterns (unused attributes vs test lists), regression workflow |

## Contributing

Open an issue or pull request.

## License

MIT
