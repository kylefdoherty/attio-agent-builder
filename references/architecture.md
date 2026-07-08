# Workflow Architecture

## Building Blocks

An Attio workflow chains blocks together. The AI-relevant blocks are:

| Block | Cost | Can Do | Limitations |
|-------|------|--------|-------------|
| **Custom Agent** | Dynamic (varies by task complexity) | Full prompt, model selection, web access, tool access, structured output | Most expensive |
| **Prompt Completion** | Cheap (fixed) | Custom prompt on record data | No web, no structured output, no model selection |
| **Classify Text** | Cheap (fixed) | Map text → select options | No custom prompt |
| **JS Code Block** | Free | Deterministic logic, mappings, transforms | No AI reasoning, no web |
| **If/Else** | Free | Route workflow based on field values | — |
| **Update Record** | Free | Write fields back to the record | — |
| **Find List Entries** | Free | Look up records in a list | — |

## How Many Agents?

The job to be done determines how many agents you need. Don't default to one — ask what
the workflow actually requires.

### Splitting Criteria

Split into multiple agents when:

- **Different tool needs.** One step needs web access, another doesn't. One needs call
  recordings, another needs emails. Separate agents can have different tool configs.
- **Different models.** A research step benefits from a more capable model, but a
  classification step works fine on a cheaper one.
- **Independent iteration.** You want to tweak classification rules without re-running
  research. Or adjust scoring without re-running enrichment. Separate agents decouple
  the iteration cycles.
- **Gating.** A cheap step can filter out records before an expensive step runs. No point
  running deep research for a company that doesn't qualify.
- **Reuse.** The output of one step feeds multiple downstream agents. Run research once,
  classify and score separately.
- **Prompt size.** The combined instructions are too long or complex for one prompt. Splitting
  reduces cognitive load on the model and makes each prompt sharper.

Keep as a single agent when:
- The task is simple and self-contained
- Splitting would just add cost with no iteration or gating benefit
- The steps are tightly coupled and don't make sense independently

### Example: Industry Classification Workflow (5 blocks)

```
Trigger: Company record created
  → Custom Agent: Research company (web access ON)
      Visits website, collects signals, validates company identity
  → Custom Agent: Classify industry (web access OFF)
      Receives research output, picks segment/industry/subsegment
  → JS Code Block: Deterministic mapping
      Maps any post-processing logic that doesn't need AI
  → If/Else: segment = "Tech"?
  → YES → Custom Agent: Deep company research
  → NO → Update Record: Write classification fields only
```

Why 3 agents + 2 utility blocks instead of 1 agent:
- Research and classification iterate independently
- Classification rules changed 10+ times without re-running web research
- Deep research only runs on qualified companies (gating saves cost)
- Segment mapping is deterministic — no reason to use AI for a lookup table

### Example: Simple Enrichment (1 block)

```
Trigger: Company record created
  → Custom Agent: Extract description, HQ, founding year
  → Update Record
```

One agent is fine here — it's a simple extraction, no gating needed, no complex classification
to iterate on separately.

## Composing Blocks

### Agent → Agent

Output from one Custom Agent is available as a variable in the next block. Name your blocks
clearly — the block name becomes the variable name.

```
Block "Research Company" → output available as {{research_company}}
Block "Classify Industry" → prompt uses {{research_company}} as input
```

### Agent → Code Block

Use when the agent produces a value that needs deterministic transformation.

**Why not do the mapping in the agent prompt?** AI models can get lookup tables wrong, and
you're paying for deterministic work. A JS code block is free, fast, and testable.

```
Custom Agent → returns industry string (e.g., "Software Development")
JS Code Block → maps industry → companySegment (e.g., "Tech")
```

### Agent → If/Else → Agent

Gate pattern. Cheap step decides whether to run expensive work.

```
Custom Agent: Classify
  → If/Else: qualifies?
    → YES: Custom Agent: Deep research
    → NO: Stop
```

## Cost Awareness

Every Custom Agent step costs credits. Don't split into multiple agents unless the split
provides a concrete benefit — iteration independence, gating, different tool needs, or
different model requirements. Splitting without a reason just multiplies cost.

Web access increases cost — only enable it when the agent needs to visit external sources.
