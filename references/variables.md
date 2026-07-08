# Attio Variable Syntax

## Format

Attio uses double-brace syntax with dot notation:

```
{{object.attribute}}
```

Examples:
- `{{company.name}}`
- `{{company.domains}}`
- `{{person.email_addresses}}`

## Placement: Variables Go at the Bottom

Put all variable injections in a single block at the end of the prompt. Don't scatter them
inline throughout the instructions.

**Why:**
- The static instructions form a consistent prefix across all runs. If the model provider
  supports prompt caching, this prefix gets cached and you only pay full price for the
  variable content that changes per record.
- Cleaner prompt structure — instructions are instructions, data is data.
- Easier to see at a glance which variables the agent needs.

**Pattern:**
```
[All static instructions, rules, examples, output format...]

COMPANY TO RESEARCH

Company Name: {{company.name}}
Domain: {{company.domains}}
```

**Anti-pattern:**
```
Visit {{company.domains}} and research {{company.name}}. Check if {{company.name}} matches
the website. Use {{company.domains}} to verify...
```

## Common Company Attributes

| Variable | What It Returns |
|----------|----------------|
| `{{company.name}}` | Company name |
| `{{company.domains}}` | Company domain(s) |
| `{{company.description}}` | Company description (if populated) |
| `{{company.categories}}` | Attio's built-in broad categories (multiselect) |
| `{{company.employee_range}}` | Employee count range |
| `{{company.funding_raised}}` | Total funding raised (if populated) |

Custom attributes use their `api_slug`:
- `{{company.industry_v2}}` (custom attribute)
- `{{company.company_segment}}` (custom attribute)

## Common Person Attributes

| Variable | What It Returns |
|----------|----------------|
| `{{person.name}}` | Full name |
| `{{person.email_addresses}}` | Email addresses |
| `{{person.job_title}}` | Current job title |

## Referencing Previous Workflow Steps

When a Custom Agent follows another block in the workflow, you can reference the previous
block's output as a variable. The exact variable name depends on what you named the block.

Example in a multi-agent workflow:
- Block named "Research Company" → output available as `{{research_company}}`
- Next block's prompt references: `{{research_company}}` to get the full output
