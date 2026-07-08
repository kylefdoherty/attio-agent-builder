# Prompt Engineering Patterns for Attio Agents

Patterns discovered through real-world iteration and published research. Each one fixes a
specific failure mode. Sources and dates are noted so patterns can be updated as the field
evolves.

---

## Structured Output Before Decisions

**Problem:** When a model needs to compare two values or evaluate evidence before making a
decision, it hallucinates the comparison if it reasons internally.

**Fix:** Force the model to OUTPUT the evidence as structured fields BEFORE the decision field.
Field ordering in the JSON matters — put evidence fields first. Autoregressive models generate
tokens left-to-right, so the order fields appear in the output directly affects accuracy.

**Example:**
```json
{
  "softwareProductSignalCount": 4,
  "softwareProductSignals": ["Login button", "Pricing page", "API docs", "Free trial CTA"],
  "primaryOfferingType": "software_product"
}
```

The model writes `softwareProductSignalCount: 4` and then can't output
`primaryOfferingType: "professional_service"` — it would contradict its own evidence.

**Anti-pattern:** Putting the decision field first and evidence after. The model picks the
decision, then selectively gathers evidence to justify it.

_Source: Own iteration — fundraising classifier v3.0 domain check fix, 2026-02. Confirmed by Anthropic structured output docs and arxiv research on prompt complexity diluting structured reasoning, 2025-2026._

---

## Consistency Rules

**Problem:** The model's reasoning can contradict its classification fields. It says "this is
a VC firm" in reasoning but outputs `segment: "Tech"`.

**Fix:** Add explicit hard constraints that tie fields together:

```
Consistency rules — hard constraints, fix violations before responding:
1. If segment is "VC", industry MUST be from the Financial Industries list
2. If segment is "Tech", industry MUST be from the Technology Industries list
3. If status is "unresolved", industry MUST be "Unclassified"
4. Every field must be a non-empty string
```

Call them "hard constraints" and tell the model to "fix violations before responding."

_Source: Own iteration — fundraising classifier, industry classifier edge cases, 2026-02._

---

## Sequential Steps Over Parallel Signals

**Problem:** Giving the model 5 parallel signals and saying "weigh them" leads to cherry-picking.
It picks signals that justify the answer it already wants.

**Fix:** Use sequential steps where each step gates the next:

```
Step 1: Visit the website → collect evidence
Step 2: Validate company identity → if mismatch, stop
Step 3: Count software signals → if 2+, classify as software_product, stop
Step 4: Only if Step 3 didn't fire, evaluate other offering types
```

_Source: Own iteration — fundraising classifier v2.6 restructure, 2026-02._

---

## Redundant Rules From Different Angles

**Problem:** A single rule can be skipped. The model finds a way around it.

**Fix:** Encode critical constraints multiple ways. Each rule produces the same outcome but
from a different entry point:

```
Rule 6: If entityType is not "company", return null
Rule 7: If fundingType is "government_budget", return null
Rule 8: If PitchBook profile type is "Investor", return null
```

All three prevent non-company entities from getting funding data, but from different angles.
The model would have to violate ALL of them to slip through.

_Source: Own iteration — funding research v4.1, Orbit Ventures and CTECS failures, 2026-03._

---

## Force Evidence Into Output Structure

**Problem:** Instructions like "check for X signals first" get ignored.

**Fix:** Make the signal count a required output field. The model must write it down before
the decision field.

```
"softwareProductSignalCount": 4,  ← model must commit to a number
"primaryOfferingType": "software_product"  ← can't ignore the count it just wrote
```

**Corollary:** If you tell the model to "check for investor signals" as an instruction, it
skips the check. If you make `investorSignals` a required output field (even as an empty
array), it has to actually look.

_Source: Own iteration — industry classifier v2, ESAI edtech signals not detected, 2026-04._

---

## Scoped Biases

**Problem:** "Err toward X when uncertain" without conditions causes overcorrection. The model
applies the bias everywhere, not just the uncertain cases.

**Fix:** Always scope biases with conditions:

```
WRONG: "Err toward software_product when uncertain"
RIGHT: "Err toward software_product WHEN software signals are present but the vertical
        is ambiguous. Err toward professional_service WHEN the company describes doing
        work for clients."
```

_Source: Own iteration — fundraising classifier v3.1, "err toward direct_fundraise" overcorrection, 2026-03._

---

## Prose Edge Cases Get Skipped

**Problem:** When an edge case is described as prose ("if the company IS an investment firm,
return null"), the model skips it in practice.

**Fix:** Make the edge case a required output field with consistency rules:

```
WRONG: "If this is an investment firm, don't classify it as Tech"

RIGHT: Add entityType as a required field:
  "entityType": "investor"  ← model must commit
  Consistency rule: if entityType ≠ "company" → segment must be "VC" or "Other"
```

_Source: Own iteration — funding research v4.1, Orbit Ventures VC portfolio and CTECS government budget failures, 2026-03._

---

## Don't Over-Specify Keywords

**Problem:** Listing specific words ("raises", "secures", "closes") makes models too literal.
They miss synonyms and paraphrases.

**Fix:** Use category descriptions:

```
WRONG: Look for words like "raises", "secures", "closes", "announces funding"
RIGHT: Look for any language indicating the company is receiving investment capital
```

_Source: Own iteration — fundraising classifier, Etched "funding boost" misclassification, 2026-02._

---

## Stopping Conditions for Research Agents

**Problem:** Research agents over-explore. They visit 10+ pages, follow every link, and burn
through credits when they already had enough information after page 2.

**Fix:** Add an explicit stopping condition to the prompt:

```
After each page you visit, ask: "Can I answer the core request now with useful evidence?"
If yes, stop researching and produce your output. Do not visit additional pages for marginal
improvements.
```

This is especially important for agents with web access on more capable models, which will
default to thorough research unless told when to stop.

_Source: OpenAI GPT-5.5 prompt guidance, 2026. Also observed in own testing — Opus 4.8 deep research burning excessive credits, 2026-05._

---

## Decision Rules Over Absolute Commands

**Problem:** `ALWAYS`, `NEVER`, and `MUST` in prompts can cause over-attendance — the model
fixates on those instructions at the expense of other context. Newer models follow
instructions more literally, which makes absolute commands riskier.

**Fix:** Use conditional decision rules instead:

```
WRONG: "NEVER classify fintech companies as Financial Services"
RIGHT: "When a company builds software that sells TO financial institutions, classify as
        Tech. When a company IS a financial institution (holds licenses, manages funds),
        classify as Financial Services."
```

The decision rule gives the model criteria to evaluate rather than a blanket prohibition
to memorize.

**Exception:** Consistency rules ("if X then Y MUST be Z") are fine as absolute constraints —
they're logical invariants, not behavioral guidance.

_Source: OpenAI GPT-4.1 prompting guide and GPT-5.5 prompt guidance, 2025-2026._

---

## Repeat Critical Rules at the End of Long Prompts

**Problem:** In long prompts, model attention is strongest at the beginning and end.
Instructions in the middle get less attention.

**Fix:** For prompts over ~500 words, repeat the most critical rules near the end, just
before the variable block. Don't repeat everything — just the rules that matter most.

```
[Main instructions...]

[Edge cases, examples, details...]

REMINDERS (critical rules):
- softwareProductSignalCount MUST appear BEFORE primaryOfferingType
- Every field must be a non-empty string or number. Never null.
- Return ONLY a single valid JSON object.

COMPANY TO RESEARCH
Company Name: {{company.name}}
Domain: {{company.domains}}
```

_Source: OpenAI GPT-4.1 prompting guide — "place instructions at both beginning AND end of long context", 2025._

---

## Lean on Examples Over Exhaustive Edge Case Lists

**Problem:** Long lists of edge case prose consume context without proportional benefit. The
model can't hold 20 edge case paragraphs in working memory equally.

**Fix:** 3-5 diverse, well-chosen examples teach more than pages of edge case documentation.
Each example should demonstrate a different decision boundary the model needs to learn.

```
LESS EFFECTIVE:
- If the company is a neobank, classify as Tech not VC
- If the company is a payment processor, classify as Tech not Financial Services
- If the company is a lending platform, classify as Tech not Financial Services
- If the company builds trading software, classify as Tech not Financial Services
- [12 more similar rules...]

MORE EFFECTIVE:
Example 4 — Fintech Software:
{"primaryOfferingType":"software_product","industry":"Software Development",...}
Reasoning: Builds payment technology (APIs, SDKs). Software vendor, not a financial institution.

Example 5 — Actual Bank:
{"primaryOfferingType":"financial_service","industry":"Banking",...}
Reasoning: IS a bank. Holds licenses, manages deposits. Not a software company.
```

If you have edge cases that examples can't cover, keep the prose rules — but try to
consolidate related rules and express them as decision criteria, not individual case lists.

_Source: Anthropic engineering blog, "Effective context engineering for AI agents", 2026._

---

## JSON Compliance Block

**MANDATORY** for any Attio Custom Agent returning structured output. Paste this at the end
of every prompt that returns JSON:

```
Return ONLY a single valid JSON object.
No markdown code fences.
No explanation before or after.
No notes. No second attempts.
If you are unsure about a field, make your best decision and explain in the reasoning
field — do not add commentary outside the JSON.
Do not output multiple JSON objects or revise your answer — commit to one response.
```

**Why this matters:** Some models add preamble text before JSON, wrap it in markdown code
fences, or output multiple JSON objects with notes between them. Any of these break Attio's
JSON parser and return null for all fields. If structured output is returning null, check
whether the model is adding extra text around the JSON — switching models often fixes it.

_Source: Own iteration — Sonnet 4.6 JSON compliance failure in Attio industry classifier, 2026-06._

---

## Null Prevention

Attio does not handle null gracefully in structured output. Always include:

```
IMPORTANT: Every field must be a valid string or number. Do not use JSON null for any field.
If a value is unknown, use "Unknown" for text fields or 0 for numbers.
```

_Source: Own iteration — Attio industry classifier, null fields breaking workflow routing, 2026-06._

---

## Field Naming

Use **camelCase** for all field names in the output schema. This is consistent with Attio's
API conventions and JavaScript code blocks that consume the output.

```
GOOD: companyDescription, softwareProductSignalCount, industryConfidence
BAD: company_description, software_product_signal_count, industry_confidence
```

_Source: Attio API conventions, 2026._
