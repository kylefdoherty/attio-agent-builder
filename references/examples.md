# Working Agent Examples

Complete agents showing the paste-ready output format. Each example shows the 5 config
fields matching the Attio UI order: Prompt, Output Schema, System Prompt, Model, Tools.

---

## Example 1: Company Research Agent

Visits a company's website and extracts structured research data — what they do, what signals
are present, and whether the company identity matches. Designed as the first agent in a
multi-agent workflow where downstream agents consume its output.

### PROMPT

```
Visit this company's website and research what they do. Collect structured evidence.

INSTRUCTIONS

Step 1: Visit the Website

Visit the company's website and research thoroughly. Look for TWO things on every page:

A) What the company does — their business, products, services

B) Software product signals — actively look for ALL of these:
- Login / Sign In button or link (check header, nav, footer)
- "Sign up", "Start free trial", "Get started" CTA
- App subdomain (e.g., app.company.com)
- Pricing page
- Product screenshots or UI demos
- API docs, developer docs, integration pages
- Named software tools or features
- AI-powered tools users interact with

Pages to check: Homepage, About, Products/Services, nav menus, footer links.

Step 2: Validate Company Identity

Check that the website belongs to the expected company. Set nameMatch to one of:
- "true" — Name matches exactly or is a clear variation
- "partial" — Names share a key word and the domain matches, or clear rebrand
- "false" — Completely different company

Step 3: Collect Evidence

Count software product signals. Critical distinction:

Count as software signals (must be products THIS COMPANY builds and operates):
- Login for THIS company's product
- "Start free trial" CTA for THIS company's product
- Public pricing page with tiers
- Product screenshots showing software UI
- API docs or integration pages
- Named product features or tools
- Self-serve onboarding
- AI-powered tools or platform

Do NOT count:
- Links to OTHER companies (portfolio, partners, subsidiaries)
- Software products built by companies they INVEST IN or OWN
- Client logos or partner logos
- Login buttons for third-party tools

Step 4: Cross-Check — Investor or Builder?

BEFORE counting software signals, check for investment/holding company language:
- "Provide capital", "invest in", "partner with founders", "acquire", "fund"
- Software listed on site are things they INVESTED IN, not BUILT
- Describes themselves as a "group", "holding company", "fund", "capital", "ventures"

If YES, softwareProductSignalCount is 0.

Company Name: {{company.name}}
Domain: {{company.domains}}

Return ONLY a single valid JSON object. No markdown code fences. No explanation before or after.
Start your response with { and end with }.
```

### OUTPUT SCHEMA

```json
{"expectedCompany":"Acme Corp","websiteCompany":"Acme Corp","nameMatch":"true","companyDescription":"Cloud monitoring platform for infrastructure, applications, and logs.","softwareProductSignalCount":5,"softwareProductSignals":["Login button in header","Free trial CTA","Public pricing page","Product screenshots","API docs"],"investorSignals":[],"whatTheyDo":"Acme Corp provides a cloud monitoring and security platform. They sell SaaS tools for DevOps teams.","howTheySell":"PLG with free trial and public pricing. Enterprise tier with Contact Sales.","whoTheySellTo":"DevOps teams, engineering organizations, enterprise IT."}
```

### SYSTEM PROMPT

Not needed — all instructions are in the prompt.

### MODEL

GPT-5.5. Clean JSON output, good accuracy on signal detection.

### TOOLS

- Web access: ON
- Tool access: None
- Connected apps: None

---

## Example 2: Classification Agent (Operates on Previous Agent's Output)

Receives research data from a previous agent and classifies the company. No web access —
works entirely on the data passed in. This is the kind of agent where the system prompt
carries the heavy logic (rules, consistency constraints, industry lists).

### PROMPT

```
Classify this company based on the research data below.

First, check the research data. If nameMatch is "false" or all fields are "Unknown", set
status to "unclassified" and use "Unclassified" for segment, subsegment, and industry.

Write a companyDescription in 1-2 sentences. Sentence one: what they do and what they sell.
Sentence two (optional): who they sell to if notable. Be specific but not exhaustive — name
the core product, not every use case or persona.

Then pick the segment, the most specific subsegment, and the best-fit industry from the
recognized lists in the system prompt.

Research data:
{{research_company}}
```

### OUTPUT SCHEMA

```json
{"status":"classified","companyDescription":"Cloud monitoring and security platform for infrastructure, applications, and logs.","segment":"Tech","subsegment":"DevTools","industry":"Software Development","industryConfidence":9,"reasoning":"Cloud monitoring platform with 5 software signals. Clear SaaS with PLG motion."}
```

### SYSTEM PROMPT

```
You are a company classification agent. You receive structured research data about a company
and classify it into a segment, subsegment, and industry. You do not visit websites or do
research — you classify based solely on the data provided.

Segment rules — apply in order:
1. If investorSignals is non-empty AND softwareProductSignalCount is 0 → segment is "VC"
2. If softwareProductSignalCount >= 2 AND the company builds the software → segment is "Tech"
3. Everything else → segment is "Other"

Edge cases that override your instincts:
- Fintech companies that BUILD software (payment APIs, banking platforms) = "Tech"
- Neobanks with software products = "Tech"
- Accelerators/incubators that INVEST in startups = "VC"
- Agencies, consulting firms, MSPs with client portals = "Other"
- Vertical SaaS (MarTech, HRTech, LegalTech, HealthTech) = "Tech"

Consistency rules — hard constraints, fix violations before responding:
1. If segment is "VC", industry MUST be from the Financial Industries list
2. If segment is "Tech", industry MUST be from the Technology Industries list
3. If segment is "Other", industry should NOT be from the Technology or Financial lists
4. Subsegment must match segment
5. If nameMatch was "false" or all fields are "Unknown": status = "unclassified", all fields = "Unclassified", confidence = 1
6. Every field must be a non-empty string

Subsegment options:
Tech: AI/ML, DevTools, FinTech, Crypto/Blockchain, Cybersecurity, Data/Analytics, Vertical SaaS, Hardware, Marketplace, E-commerce Tech, EdTech, Media Tech, Telecom, Other Tech
VC: Venture Capital, Private Equity, Accelerator/Incubator, Angel/Family Office, Other Investment
Other: Agency/Consulting, MSP/IT Services, Healthcare, Financial Services, Manufacturing, Real Estate, Retail/E-commerce, Education, Media/Publishing, Recruitment, Legal, Nonprofit, Construction, Energy, Unclassified

Industry lists:
Financial Industries: Venture Capital and Private Equity Principals, Investment Management, Investment Banking, Capital Markets, Banking, Insurance, Financial Services
Technology Industries: Software Development, Computer and Network Security, Data Infrastructure and Analytics, Artificial Intelligence (AI) & Machine Learning (ML), Blockchain Services, Biotechnology Research, Telecommunications, Semiconductor Manufacturing, Computer Hardware
Other Industries: Any standard industry name not in the two lists above.

Confidence scale:
- 9-10: Obvious, clear evidence
- 7-8: Strong match, minor ambiguity
- 5-6: Likely correct, some interpretation
- 3-4: Best guess, limited evidence
- 1-2: Very little data or mismatch
```

### MODEL

GPT-5.5. Could test GPT-5 mini for cost savings — this agent doesn't need web access or
complex reasoning, just rule application.

### TOOLS

- Web access: OFF
- Tool access: None
- Connected apps: None

---

## Example 3: Data Validation Agent

Checks whether a company's LinkedIn URL, name, and domain all refer to the same entity.
Catches bad CRM data before downstream agents run on it.

### PROMPT

```
Check whether this company's LinkedIn profile, name, and domain refer to the same entity.

1. Visit {{company.domains}} and note the company name on the website
2. Visit the LinkedIn URL and note the company name and website listed there
3. Compare all three: CRM name, website name, LinkedIn name/domain

Company Name: {{company.name}}
Domain: {{company.domains}}
LinkedIn URL: {{company.linkedin_url}}

Return ONLY a single valid JSON object. No markdown code fences. No explanation before or after.
```

### OUTPUT SCHEMA

```json
{"crmName":"Acme Corp","websiteName":"Acme Corp","linkedinName":"Acme Corporation","linkedinDomain":"acme.com","nameMatch":true,"domainMatch":true,"isValid":true,"mismatchDetails":"","confidence":9}
```

### SYSTEM PROMPT

Not needed — straightforward comparison task.

### MODEL

GPT-5.5.

### TOOLS

- Web access: ON
- Tool access: None
- Connected apps: None
