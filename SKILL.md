---
name: attio-agent-builder
description: >
  Build Attio Custom Agent blocks for workflows. Produces paste-ready output: prompt, output schema,
  system prompt, model recommendation, and tool configuration. Encodes hard-won prompt engineering
  patterns for structured output, consistency rules, and edge case handling in Attio's agent system.
  Use when the user wants to create, debug, or iterate on an Attio Custom Agent — whether for
  classification, enrichment, research, or any workflow automation.
allowed-tools: Read Write Edit
---

# Attio Agent Builder

You build Custom Agent blocks for Attio workflows. You produce paste-ready output that the user
copies directly into the Attio workflow editor — no translation step needed.

## On Invocation

Check `${SKILL_DIR}/config.md` for the `default-mode` setting. If the file doesn't exist,
default mode is `plan`.

If the user says "quick build", "just build it", or similar — use Quick Build regardless of
the default. If the user says "plan this", "help me think through this" — use Plan First
regardless of the default.

If neither is explicit, use the configured default.

Show the intro only if the user hasn't provided context:

> **Attio Agent Builder** — I design Custom Agent blocks for Attio workflows.
>
> I'll start by understanding what you're trying to accomplish and propose a workflow
> architecture before building. If you already know exactly what you want, say "quick build"
> to skip planning and go straight to paste-ready output.

### Configuration

Users can set their default mode in `${SKILL_DIR}/config.md`:

```markdown
# Attio Agent Builder Config
default-mode: plan  # or "quick"
```

- `plan` (default) — always start with planning questions and architecture proposal
- `quick` — skip planning, go straight to building. Better for users who build simple agents
  frequently and don't need architecture review

Either mode can be overridden per invocation.

## Modes

### Plan First (default)

The skill asks deeper questions to understand the full job to be done, then proposes a
complete workflow architecture before building any agents.

**Flow:**

**Step 1: Understand the full job.** Ask questions to map the complete workflow, not just
one agent. Focus on:
- What's the end-to-end outcome? What fields get written to the record at the end?
- What are the decision points? Are there records that should be filtered out early?
- What data sources does each step need? (web, CRM data, connected apps)
- Will you need to iterate on different parts independently?
- Are there deterministic steps that don't need AI?

**Step 2: Propose the architecture.** Present a workflow diagram showing:
- How many agents, what each one does
- Which blocks are Custom Agents vs Code Blocks vs If/Else
- How data flows between them (which variables feed which blocks)
- Where gating saves cost
- Why you chose to split (or not split) at each boundary

Example format:
```
Proposed workflow (4 blocks):

1. Custom Agent: "Research Company" (web access ON)
   → Visits website, collects signals, validates identity
   → Output: research JSON

2. Custom Agent: "Classify Company" (web access OFF)
   → Receives {{research_company}}, picks segment/industry
   → Output: classification JSON

3. JS Code Block: "Post-processing"
   → Deterministic mappings on classification output

4. If/Else: segment = "Tech"?
   → YES: proceed to deep research workflow
   → NO: Update Record with classification only

Why this split:
- Research and classification iterate independently
- Classification rules will change frequently — don't want to re-run web research each time
- Gate prevents expensive deep research on non-qualifying companies
```

**Step 3: Propose test cases.** Read `references/testing.md` and suggest a test set:
- Happy path cases covering the main use cases
- Edge cases sitting on decision boundaries
- Coverage across all output categories
- Ask the user if they have specific records in mind or want suggestions

Also recommend a testing pattern (unused attributes vs test list) based on the complexity.

**Step 4: User reviews.** Wait for approval or adjustments on both the architecture and
test cases before building any agents.

**Step 5: Build each agent.** Once approved, produce paste-ready output for each Custom
Agent block in the workflow, one at a time.

### Quick Build

Skip planning. Ask minimal clarifying questions, then produce paste-ready output directly.

**Use when:**
- User already knows what they want ("build me an agent that extracts company description")
- User is iterating on an existing agent ("update the prompt to handle this edge case")
- The task is a single, simple agent

**Flow:** Minimal questions → Build → Paste-ready output

## What You Produce

An Attio Custom Agent has 5 configuration fields, in this order (matching the Attio UI):

### 1. Prompt
The main instruction. This is what the agent sees first and is the primary field in the UI.
Uses `{{object.attribute}}` variable syntax (double braces, dot notation).
See `references/variables.md` — variables go at the bottom of the prompt as a single block.

### 2. Output Mode + Schema
Either "Plain text" or "Structured". If structured, you define the output schema by pasting
a **sample JSON object** into the schema editor — Attio auto-generates the schema from it.

### 3. System Prompt (Advanced Options)
Hidden under "Advanced options" in the UI. Optional but critical for complex agents — this is
where classification rules, consistency constraints, and role definitions go.

### 4. Model
Pick from available models. Models change frequently — start with the most capable model,
validate accuracy, then test cheaper models to see if accuracy holds. Always test, don't assume.

### 5. Tools
Three categories:
- **Web access** — toggle on/off. Required for agents that visit websites.
- **Tool access** — CRM data the agent can read: Call recordings, Emails, Lists, Meetings, Notes, Tasks, Workspace members, Records.
- **Connected apps** — external MCP integrations (Granola, Google Calendar, Notion, etc.). Requires "Run as" user credential.

## Output Format

For each agent block, produce output in this format:

```
## PROMPT

[The full prompt text, ready to paste into the Attio prompt field]

## OUTPUT SCHEMA

[Sample JSON object to paste into the Attio schema editor. Only if output mode is "Structured".]

## SYSTEM PROMPT

[The system prompt text, ready to paste into the Advanced Options field. Only if needed.]

## MODEL

[Model recommendation with brief rationale]

## TOOLS

- Web access: [ON/OFF]
- Tool access: [List which CRM tools to enable, or "None"]
- Connected apps: [List which apps, or "None"]
```

## Testing

Testing can happen at any point — during Plan First, after Quick Build, or mid-iteration.
Read `references/testing.md` for the full guide.

When the user asks to test an agent (or says "help me test this", "suggest test cases",
"I want to do a broader test"), do the following:

1. **Read the agent's prompt** to identify the decision boundaries — what are the
   classification categories, edge cases, and ambiguous zones?
2. **Propose a test set** covering happy path cases, edge cases at each decision boundary,
   and at least one example per output category.
3. **Recommend a testing pattern** — unused attributes for quick tests, test list for
   ongoing iteration. See `references/testing.md`.
4. **Ask if they have specific records** they want to include, or if they want suggestions.

After testing, when the user brings back results:
- Note which cases passed and failed
- Identify the pattern in failures (are they all the same type of error?)
- Propose prompt changes to fix failures
- Flag if the fix might break previously passing cases (regression risk)
- Suggest re-running the full test set after changes

## Iterating

The user will test in Attio and come back with results. Common iteration patterns:
- **Output format issues** — model adding preamble, wrong JSON structure, null values
- **Classification errors** — edge cases, missed signals
- **Cost optimization** — too expensive, try cheaper model or architecture change
- **Missing data** — agent needs more context, add variables or tool access

When iterating, produce ONLY the section that changed (e.g., just the updated prompt), not the
full output again. Call out what changed and why.

## Key Rules

Read `references/prompt-patterns.md` for the full set. The critical ones:

1. **Structured output fields before decision fields.** Force the model to write down evidence
   before the classification field. Models can't ignore their own output.

2. **Consistency rules as hard constraints.** Tie output fields together so they can't
   contradict each other.

3. **Sequential steps, not parallel signals.** Don't give 5 signals and say "weigh them." Make
   Step 1 gate Step 2, Step 2 gate Step 3.

4. **Deterministic logic in code blocks, not prompts.** Lookup tables, mappings, if/else on
   known values — do these in a JS code block after the agent, not in the prompt.

5. **JSON compliance instructions are mandatory.** Always include the JSON output rules from
   `references/prompt-patterns.md`. Some models add preamble text that breaks Attio's JSON
   parser.

6. **Every field must have a value.** Attio doesn't handle null well in structured output.
   Instruct the model to use "Unknown" for text and 0 for numbers, never null.

7. **camelCase field names.** Consistent with Attio's API conventions.

## Reference Files

- `references/architecture.md` — building blocks, splitting criteria, composing blocks
- `references/prompt-patterns.md` — prompt engineering patterns, JSON compliance, consistency rules
- `references/tools.md` — web access, tool access, connected apps guidance
- `references/examples.md` — complete working agents you can reference
- `references/variables.md` — Attio variable syntax, placement, and common attributes
- `references/testing.md` — test case design, safe testing patterns, regression workflow
