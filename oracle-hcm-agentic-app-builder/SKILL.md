---
name: oracle-hcm-agentic-app-builder
description: Build import-ready Oracle Fusion 26B Agentic App ("workspace") JSON artifacts that bundle AI Agent Studio agent teams into one swimlane workspace with Summary / Actions / Communications panels. Use when asked to create, package, or debug an Agentic App / AI workspace (the Apps-tab container), to wire member agents into an app, or to author the oraInfoDisplay / oraComms / ora.Invoke output blocks that populate app panels. Complements oracle-hcm-agent-builder (which builds the agents themselves).
---

# Oracle HCM Agentic App Builder (26B)

An **Agentic App** (a.k.a. *Workspace*) is the 26B AI Agent Studio **Apps-tab** artifact that composes multiple **agent teams** into one purpose-built UI with four panels: **Information** (swimlane insight cards), **Ask Oracle** (conversational query), **Actions** (priority action buttons), **Communications** (draft emails/messages). This skill builds the **app container artifact**; build the member agents with `oracle-hcm-agent-builder`.

Everything here was reverse-engineered from **real seeded exports** on a 26B pod (`HCM_HRL_MANAGER_CONCIERGE_WORKSPACE`, `ORA_MY_HELP_WORKSPACE_FOR_EMPLOYEES`, `ORA_HCM_IRC_HIRING_MANAGER_WORKSPACE`, plus SCM/INV/maintenance apps) and the Oracle 26B PDF `how-do-i-use-ai-agent-studio.pdf` (G55148-06) + partner-training decks.

## Authoritative reference
`references/agentic_app_deck_reference.md` — the full build process, all 7 widget schemas, oraInsight/oraComms exact schemas, the OraMessageHint lifecycle, context params, action-step types, and per-agent prerequisites, **verbatim from the Oracle partner decks (May 2026) with slide cites**. Read it before building. This SKILL.md is the quick map; that file is the source of truth.

## App artifact: UI-PRIMARY build, JSON form exists
Oracle's **documented creation path is UI-only** (Deep Dive Slide 21: "All configuration through visual interfaces … No direct JSON editing required"). HOWEVER, seeded apps **export to JSON** from the pod (you have real exports), so the artifact below is a faithful serialized form. **Use the JSON as the spec to click through in the Apps tab** (and document every Builder field for Marketplace re-import). A hand-authored app JSON matching the seeded export shape MAY import as a shortcut, but treat that as unverified — the agents import as JSON for certain; the app shell is rebuilt in the UI.

## App artifact schema (verbatim shape from seeded exports)
Top level — EXACTLY these 7 keys:
```json
{
  "name": "Compass Steward Workspace",
  "code": "NTT_COMPASS_STEWARD_WORKSPACE",
  "specification": "<STRINGIFIED JSON — see below>",
  "internalName": "Compass Steward Workspace",
  "version": 1,
  "internalDescription": "What this workspace unifies and for whom.",
  "status": "DRAFT"
}
```
- **`specification` is a STRINGIFIED JSON string**, not a nested object. Build the spec object, then `json.dumps()` it into this field. (This trips everyone — a nested object fails import.)
- `status`: `"DRAFT"` for a new build (let the owner publish), `"PUBLISHED"` once live.
- Write the file with a **UTF-8 BOM** (Studio exports carry one): `f.write('﻿')` then dump.

### specification.applicationMetadata (the parsed spec)
```json
{
  "applicationMetadata": {
    "title": "Compass Steward",
    "subTitle": "HR Shared Services action workspace",
    "subtitleAgentCode": "<an agent code, drives the dynamic subtitle>",
    "pagePattern": "swimlanesPattern",
    "pageConfig": {
      "agentContainers": [
        {"id": "container-cs-1", "title": "Reporting Line", "agents": ["agent-cs-1"]},
        {"id": "container-cs-2", "title": "Working Hours",  "agents": ["agent-cs-2"]}
      ],
      "layout": "2",
      "firstLane":  ["container-cs-1"],
      "secondLane": ["container-cs-2"]
    },
    "agents": {
      "agent-cs-1": {
        "agent": "NTT_MANAGER_UPDATE_AGENT_HRSS",   // the member agent's WorkflowCode/AgentCode
        "includeInSummary": true,
        "includeInActions": true,
        "includeInCommunications": true,
        "displayPrompt": ""
      }
    },
    "communications": [],
    "actions": [],
    "queryAgent": "NTT_MANAGER_UPDATE_AGENT_HRSS",   // the Ask-Oracle conversational agent
    "initiallyHideActions": "false",                  // or "all"
    "autoTranslate": false,
    "summary": {"agentCode": "<code>", "prompt": "Summarize ... in 1-2 sentences."},
    "security": {"roles": ["PER_LINE_MANAGER_ABSTRACT"]},  // optional: restrict by Fusion role
    "templates": []                                    // PPT/email comm templates (optional)
  }
}
```

### Field rules
- **`agents` map**: keys are synthetic ids (`agent-<n>`); each value's `"agent"` field is the **member agent's WorkflowCode/AgentCode** (must already exist + be published on the pod). `includeInSummary/Actions/Communications` decide which panels that agent feeds.
- **`agentContainers`** = swimlane insight cards; each references one (or more) synthetic agent id. `firstLane`/`secondLane` order the containers across two columns (`layout:"2"`). Every lane entry must be a real container id; every container's `agents` must be real agent-map ids.
- **`queryAgent`** = the conversational agent behind "Ask Oracle". **`subtitleAgentCode`** + **`summary.agentCode`** usually point at one "overview" agent. For an all-action suite with no overview agent, point them at your primary action agent (or build a thin read-only overview agent — preferred for a polished app).
- **`security.roles`** = Fusion abstract/duty roles that gate visibility (seeded Manager Concierge uses `PER_LINE_MANAGER_ABSTRACT`). Omit for unrestricted.
- **`actions`** entries (optional) are app-level events, e.g. a refresh button:
  ```json
  {"id":"<uuid>","code":"refreshFollowup","displayName":"Refresh","description":"",
   "events":{"onInvoke":[{"id":"<uuid>","type":"refreshAgents","params":{"agentCodes":["<agentCode>"]}}]}}
  ```

## Member-agent prerequisites (each agent the app references)
1. **Exposed to apps**: agent must have the **"Use in apps" / "Expose to agentic apps"** toggle ON. In the workflow-agent export JSON this is `AiAppsCompatibleFlag: true` at the root (default seeded value is `null`/`false`).
2. **Published**: only Published agents appear in App Builder / resolve in an app.
3. **App Experience tab** (per LLM/agent node): enable the **widgets for Actions** and/or **Communications** you want that agent to feed.
4. **Communication templates**: if using the Communications panel, define email/PPT/text templates in App Builder first; the agent references them by id.

## Output blocks the agent emits (to populate panels)
Emitted as **raw text in the terminal LLM node's response** (the app runtime parses them out). Branch the workflow on **`OraMessageHint`**:

| OraMessageHint | Agent returns |
|---|---|
| `Summary` | short summary string |
| `InitSubtitle` | dynamic subtitle string |
| `InitDisplay` | one or more `oraInfoDisplay` blocks |
| `InitActions` | zero+ action suggestions |
| `InitCommunications` | zero+ `oraComms` blocks |
| `Query` | answer text (+ optional displays/actions/comms) |
| `InvokeAction` | handle action; payload in `$context.$app.$OraAction` |
| `FillParameters` | fill comm-template params (`$context.$app.$OraCommParamsToFill`) |

**`oraInfoDisplay`** (Information widget) — verbatim from the 26B PDF:
```
<oraInfoDisplay key="maintenance-card">
{ "patternId":"cardWidget",
  "config":{ "subject":"Maintenance Complete","summary":"...","badgeText":"Info","variant":"info",
             "link":{"text":"View Details","action":"ora.Invoke(\"viewSummary\")"} } }
</oraInfoDisplay>
```
Widget `patternId`s: `cardWidget`, `multiRecordWidget` (table with row `action`/`drillDownAction`), `chartWidget` (has an `insights[]` array), etc. Action commands inside widgets: **`ora.Invoke("actionCode", payload)`**.

**`oraInsight`** (Actionable Insight — Actions panel) — REAL tag, emitted on `InitActions` (confirmed in the partner Deep Dive deck, Slides 49-51):
```
<oraInsight>
{ "title":"Update Reporting Line - Jane Doe","shortDescription":"Manager change pending, effective 2026-07-01.",
  "actionText":"Initiate Change","followUpCommand":"ora.Invoke(\"updateReportingLine\", \"{\\\"employeeId\\\":\\\"123\\\"}\")","priority":true }
</oraInsight>
```
`followUpCommand` must be `ora.Invoke("action","ctx?")` | `ora.Agent(content)` | `ora.App.*`. Enable via 3 points: LLM node Output Tab "Enable Actions for the Widgets" + App Builder "Include in Actions" + the workflow `InitActions` branch emitting the block.

**`oraComms`** (Communications draft — Email/Text), emitted on `InitCommunications`:
```
<oraComms>
{ "type":"email","title":"Reporting Line Change","content":"Dear Manager,\n\n...(REAL content, no placeholders)...",
  "params":[{"id":"email","defaultValue":"manager@company.com"},{"id":"subject","defaultValue":"..."}] }
</oraComms>
```
Text type adds `"targetAgent":"<agentCode>"`. Lifecycle: Suggest→Prefill→Fill(`FillParameters`)→Review(user)→Send(`SendCommunication`) — same agent, different prompts.

### WAF rule for the blocks (Akamai/Kona)
The blocks contain literal `<`/`>`. They live in the **agent OUTPUT** (response body) — fine at runtime. But if you **hardcode them inside a CODE node `sourceCode`** (or any prompt) they are in the **import/publish POST** → angle-bracket XSS score → **HTTP 403** on publish (the misleading "can't exit sandbox" toast). Mitigations:
- Prefer a **terminal LLM node** instructed to emit the block (LLM generates the `<...>` at runtime; keep its prompt free of full literal tags — describe the shape or use `[...]` placeholders).
- If you must build the string in CODE, **concatenate so no literal `<tag` sequence appears in source**: `"<" + "oraInfoDisplay key=..." ` etc. Bisect-test publish with a junk POST (403 = blocked, 400 = ok) before the real PATCH.

## Build steps (UI, for reference / what the artifact maps to)
1. Each member agent: Agent Team → Details → **"Use in apps" ON** → Save → **Publish**.
2. AI Agent Studio → **Apps tab → New** → name it → add the published agent teams.
3. Assign panels (Summary/Actions/Communications) per agent; lay out swimlane containers across lanes.
4. (Optional) define Communication templates; add app-level refresh Actions.
5. Publish the app. **Importing the JSON artifact built per this skill is the offline equivalent of steps 2-4.**

## Seeded app catalog (reference patterns — study before building)
| App code | Bundles | Pattern |
|---|---|---|
| `HCM_HRL_MANAGER_CONCIERGE_WORKSPACE` | Team/Compensation/Activity/Talent/Absence **Insights** agents | swimlane insight cards; `security.roles=[PER_LINE_MANAGER_ABSTRACT]`; PPT template; one Insights agent is queryAgent+summary |
| `ORA_MY_HELP_WORKSPACE_FOR_EMPLOYEES` | Help-desk What's-Trending / Pending / Waiting agents | `initiallyHideActions:"all"`, `autoTranslate:true`, refresh actions |
| `ORA_HCM_IRC_HIRING_MANAGER_WORKSPACE` | recruiting hiring-manager agents | HCM recruiting |
| `ORA_FOM_SALES_ORDER_COMMAND_CENTER`, `ORA_INV_ITEM_AVAILABILITY_WORKSPACE`, `ORA_MAINTENANCE_WORK_ORDER_MATERIAL_SHORTAGES_APP` | SCM/INV/maintenance | cross-pillar examples |

Insight-agent apps populate swimlane cards (sum=true). **Action/transactional agents** (confirm-before-write) fit best behind **queryAgent** (Ask Oracle) + **Actions** — for a clean swimlane card, give each action agent, or a companion read-only overview agent, an `InitDisplay` path that emits an `oraInfoDisplay` status card.

## Verification checklist (do before declaring done)
- [ ] Top-level keys are exactly `{name, code, specification, internalName, version, internalDescription, status}`.
- [ ] `specification` is a **string** that `json.loads()` re-parses to `{applicationMetadata:{...}}`.
- [ ] `pagePattern` = `swimlanesPattern`; every lane id ∈ container ids; every container agent id ∈ `agents` map keys.
- [ ] Every `agents[*].agent` code is a **real, published** agent on the target pod (and `AiAppsCompatibleFlag:true`).
- [ ] `queryAgent` / `subtitleAgentCode` / `summary.agentCode` reference real agent codes.
- [ ] UTF-8 BOM written.
- [ ] No literal `<oraInfoDisplay`/`<oraComms` tags hardcoded in any member agent's CODE-node source (WAF) — generate at runtime.

## Worked example
`~/Downloads/_build_compass_steward_app.py` builds `NTT_COMPASS_STEWARD_WORKSPACE.json` (bundles 4 HRSS action agents) with a self-verify block — use it as the template builder.
