# Attio Agent Builder

A Claude Code skill for building Custom Agent blocks in Attio workflows.

## Why this exists

Building reliable AI agents takes iteration. Getting structured output right, handling edge cases, designing multi-agent workflows, and testing systematically are all hard problems regardless of platform. This skill encodes prompt engineering patterns learned from building and iterating on production Attio agents -- structured output ordering, consistency rules, sequential gating, JSON compliance, and edge case handling -- so you don't have to rediscover them from scratch.

## What this skill does

Two modes:

- **Plan First** (default) -- Asks questions to understand the full workflow, proposes an architecture with test cases, then builds each agent block.
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

## Roadmap

**Local testing loop.** Right now, iteration happens inside the Attio workflow editor -- run the agent on a record, check the output, tweak the prompt, repeat. The goal is to run and test agent prompts locally in Claude Code against a set of test cases, check outputs, and iterate until the agent is 70-90% ready. Then paste into Attio for final end-to-end testing with real web access and workflow routing. This would dramatically reduce the iteration cycles spent in the agent builder UI.

**Generate JS code blocks.** The skill currently produces Custom Agent prompts but not the JavaScript code blocks that often follow them (lookup tables, field mappings, post-processing logic). Generating both the agent prompt and its companion code block together would make the full workflow paste-ready, not just the AI parts.

## Contributing

Open an issue or pull request.

## License

MIT
