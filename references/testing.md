# Testing Attio Agents

## Designing Test Cases

When building a new agent, propose a test set that covers:

### 1. Happy Path Cases
The obvious, straightforward cases the agent will handle most of the time. These should
be easy wins — if the agent fails these, something is fundamentally wrong.

Example for an industry classifier:
- Clear SaaS company (Datadog — obvious software signals)
- Clear VC firm (a16z — obvious investment signals)
- Clear agency (small marketing shop — obvious service signals)

### 2. Edge Cases
The hard classifications where the agent is most likely to fail. These are where the real
iteration happens. Identify edge cases by asking:

- **Where are the decision boundaries?** What cases sit right on the line between two
  classifications? (e.g., fintech: is it a software company or a financial institution?)
- **What looks like X but is actually Y?** (e.g., holding company with portfolio companies
  that have software signals — looks like Tech but is actually VC)
- **What has conflicting signals?** (e.g., vertical SaaS in marketing — has both software
  signals AND marketing language)
- **What has missing or sparse data?** (e.g., company website is down, domain is parked,
  sparse single-page site)
- **What has bad input data?** (e.g., company name doesn't match the domain, LinkedIn URL
  points to wrong entity)

### 3. Coverage Across Categories
Make sure every possible output category is represented. If the agent can return 5 different
classifications, the test set should include at least one example of each.

### Test Set Size

- **Initial build:** 5-10 cases. Enough to validate the core logic and catch obvious failures
  without spending too many credits. Include 2-3 happy path + 3-5 edge cases.
- **Before deployment:** Expand to 15-25 cases. Cover all output categories and the edge
  cases you've discovered during iteration.
- **Regression:** Keep the full set and re-run when you change the prompt. Any previously
  passing case that now fails is a regression.

### How to Propose Test Cases

When the skill is in Plan First mode, propose test cases as part of the architecture plan:

```
Suggested test set (8 cases):

Happy path:
1. [Company A] — clear software company, should classify as Tech/Software Development
2. [Company B] — clear VC firm, should classify as VC/Venture Capital
3. [Company C] — clear agency, should classify as Other/Marketing Services

Edge cases:
4. [Company D] — fintech that builds payment APIs (Tech, not Financial Services)
5. [Company E] — holding company with software portfolio companies (VC, not Tech)
6. [Company F] — vertical SaaS in marketing (Tech, not agency)
7. [Company G] — neobank with SaaS wrapper (Tech, not Financial Services)
8. [Company H] — sparse/parked website (should handle gracefully)
```

Ask the user if they have specific companies in mind or if they want you to suggest ones
based on the classification boundaries.

## Safe Testing in Attio

Two patterns for testing without impacting production data:

### Pattern 1: Use Unused Attributes

Write agent output to attributes that aren't connected to any downstream workflows,
automations, or reports. This way you can run the agent on real records without affecting
anything.

**How:**
- Create new attributes for testing (e.g., `industry_test`, `segment_test`) or identify
  existing attributes that aren't being used
- Point the workflow's Update Record block at the test attributes instead of the production ones
- Run the agent, review results on the test attributes
- When satisfied, switch the Update Record block to write to the real attributes

**Pros:** Tests on real records with real data. No list management needed.
**Cons:** Clutters the data model with test attributes. Need to clean up after.

### Pattern 2: Test List with List Attributes

Create a dedicated test list and use list-scoped attributes to store agent output.
List attributes only exist on records while they're in that list, so there's zero
impact on the main record data.

**How:**
- Create a list (e.g., "Agent Testing")
- Add the records you want to test to the list
- Create list-scoped attributes for the agent's output fields (e.g., `industry_test`,
  `segment_test` as list attributes)
- Point the workflow at the list attributes
- Run, review, iterate
- When satisfied, switch to writing to real record attributes and remove the test list

**Pros:** Complete isolation. No impact on record data. Easy to see all test results in
one view. Can add/remove test records easily.
**Cons:** Slightly more setup. Need to manage list membership.

### Which to Use

- **Quick one-off test:** Pattern 1 (unused attributes) — less setup
- **Ongoing iteration with a stable test set:** Pattern 2 (test list) — better isolation
  and easier to manage
- **Multiple people testing:** Pattern 2 — everyone looks at the same list

### Testing Workflow

1. **Set up test infrastructure** (unused attributes or test list)
2. **Run the agent** on the test set
3. **Review results** — check each output against expected values
4. **Identify failures** — note which cases failed and why
5. **Iterate on the prompt** — fix failures, re-run on failing cases
6. **Regression check** — re-run on previously passing cases to make sure fixes didn't break them
7. **Expand test set** — add new edge cases discovered during iteration
8. **Switch to production** — point at real attributes, remove test infrastructure

_Source: Own iteration — industry classifier testing in Attio, 2026-06._
