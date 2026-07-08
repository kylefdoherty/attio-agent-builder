# Tools Configuration for Attio Custom Agents

Custom Agents have three categories of tools. Each is configured independently.

## Web Access

Toggle on the agent block. When ON, the agent can visit URLs, search Google, and browse
websites.

**Enable when:**
- Agent needs to visit company websites
- Agent needs to search for information not in the CRM
- Agent needs to verify or research external sources

**Disable when:**
- Agent works entirely on data already in the CRM or passed as variables
- Agent is a Stage 2 classifier operating on Stage 1 output
- You want to minimize cost

**Gotcha:** Web access may increase credit cost since the model does more work (browsing,
reading pages). Only enable it when the agent actually needs to visit external sources.

## Tool Access (CRM Data)

Checkboxes that give the agent read access to specific CRM data types:

| Tool | What It Gives the Agent | When to Enable |
|------|------------------------|----------------|
| **Call recordings** | Transcripts and metadata from recorded calls | Account research, meeting summaries, relationship context |
| **Emails** | Email threads linked to the record | Communication history, follow-up context, relationship signals |
| **Lists** | List memberships and list data | Pipeline stage, deal context, segmentation |
| **Meetings** | Meeting records and metadata | Activity history, engagement signals |
| **Notes** | Notes attached to the record | Internal context, previous research, team observations |
| **Tasks** | Tasks linked to the record | Open action items, follow-up tracking |
| **Workspace members** | Team member info | Assignment, ownership, team context |
| **Records** | Access to related records (linked companies, people, deals) | Cross-record enrichment, relationship mapping |

**Default:** Leave all off unless the agent specifically needs CRM context. Every enabled
tool increases what the model processes, which can increase cost and introduce CRM data
contamination (the model may echo back CRM field values as if they were independently
researched facts).

**CRM data contamination risk:** When an agent has tool access to Records, it sees existing
field values. If those values have errors,
the agent may treat them as ground truth. For classification/enrichment agents that should
research independently, disable Records access and pass only the specific fields you need
via variables.

## Connected Apps

External integrations via MCP. Available apps depend on what's connected to the Attio workspace.
Common examples:

| App | What It Provides | Use Case |
|-----|-----------------|----------|
| **Granola** | Meeting transcripts and notes from Granola | Detailed call context, discovery notes |
| **Google Calendar** | Calendar events and scheduling data | Meeting frequency, engagement patterns |
| **Notion** | Notion pages and databases | Internal documentation, project context |

**Requirements:**
- The app must be connected to the Attio workspace
- "Run as" must be set to a specific user with credentials for that app
- The agent prompt should reference what to look for in the connected app

**When to use:** When the agent needs context that lives outside Attio but is accessible
through a connected integration. Example: an account research agent that pulls recent meeting
notes from Granola and combines them with CRM data for a briefing.

## Tool Configuration by Agent Type

### Classification Agent (e.g., industry classifier)
- Web access: ON (needs to visit company website)
- Tool access: None (classify independently, don't echo CRM data)
- Connected apps: None

### Enrichment Agent (e.g., company research)
- Web access: ON
- Tool access: Notes (to see what's already been gathered), Records (to see related data)
- Connected apps: Granola (for meeting context)

### Internal Analysis Agent (e.g., deal scoring, account health)
- Web access: OFF (works on internal data only)
- Tool access: Call recordings, Emails, Notes, Tasks, Meetings
- Connected apps: Granola, Google Calendar

### Data Validation Agent (e.g., LinkedIn/domain match checker)
- Web access: ON (needs to verify external sources)
- Tool access: Records (to read current field values to validate)
- Connected apps: None
