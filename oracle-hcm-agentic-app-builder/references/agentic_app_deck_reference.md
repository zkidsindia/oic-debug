# Oracle Fusion 26B Agentic App — Authoritative Reference (Partner Decks, May 2026)

Verbatim from Oracle partner-training decks: DD = Agentic_Apps_Deep Dive, BP = BestPractices_Roadmap26C, OV = Overview. Slide numbers cited inline. This is the canonical build reference — prefer it over blog/PDF-only sources.

## 1. Build process (UI) — DD Slide 29
1. **Create app shell** — AI Agent Studio → "New Agentic App" → name it; opens in Builder.
2. **App Settings** (gear): Title, Subtitle (static), Dynamic Subtitle Agent (`InitSubtitle`), Security Roles, Page Layout.
3. **Ask Oracle + Summary agents** — assign the query agent (orchestrator routes by agent description) and the summary agent (`Summary` hint, 1-2 sentences).
4. **Save / Run / Iterate** — replay hints, adjust layout & panel names.
**PREREQUISITE (Slide 29):** every agent must have "Expose to Agentic Apps" enabled in its **Output Tab**.

### Builder UI elements (DD 22): header, gear=App Settings, panel area, Add Agent, Ask Oracle slot, Summary slot, Actions panel (Actions Editor), Templates panel (Template Editor), Publish, Undo/Redo, Play.

### Agent Editor per panel agent (DD 32): Agent Code (search published+exposed), Include in Summary, Initial Summary Prompt, Include in Actions, Initial Actions Prompt, Initial Graphics Display Instructions, Override widgets used, Include in Communications, Panels.

## 2. Artifact: UI-primary, but a JSON form exists
- **DD Slide 21:** "All configuration through visual interfaces ... No direct JSON editing required." Oracle's documented creation path is **UI-only**.
- **BUT** seeded apps **export as JSON** from the pod (e.g. `HCM_HRL_MANAGER_CONCIERGE_WORKSPACE_*.json`, `ORA_MY_HELP_WORKSPACE_FOR_EMPLOYEES.json`) — top-level `{name, code, specification:"<stringified JSON>", internalName, version, internalDescription, status}`; spec = `applicationMetadata{title, subTitle, subtitleAgentCode, pagePattern:"swimlanesPattern", pageConfig{agentContainers, layout, firstLane, secondLane}, agents{<id>:{agent:CODE, includeInSummary/Actions/Communications, displayPrompt}}, communications, actions, queryAgent, initiallyHideActions, autoTranslate, summary{agentCode,prompt}, security{roles[]}, templates[]}`.
- **Reconciliation:** build/maintain the four agents as JSON (importable). The app shell is built in the UI; a hand-authored app JSON matching the seeded export shape MAY import as a shortcut but is not the documented path — treat it as the spec to click through, and document every Builder field for reproducibility (Marketplace re-import recreates the app shell manually).

## 3. App lifecycle hints — OraMessageHint (OV 11 / DD 12)
Init order on app load: **InitSubtitle → Summary → InitDisplay → InitActions → InitCommunications**. Runtime: **Query** (user types Ask Oracle), **InvokeAction** (clicks action; payload in `$OraAction`), **FillParameters**, **SendCommunication**, **AdditionalContent**. Workflow branches on `{{$context.$app.$OraMessageHint}}` via a Switch node; the SAME agent handles all stages with different prompts.

## 4. Output blocks (DD 46) — THREE real tags
| Output | Tag | Required fields |
|---|---|---|
| Information Display | `<oraInfoDisplay>` | key, patternId |
| Actionable Insight | `<oraInsight>` | title, shortDescription, followUpCommand |
| Communication | `<oraComms>` | type, content |

### 4.1 oraInfoDisplay widgets (DD 38-45) — outer: `{title, patternId, description, properties}`
- **chartWidget**: properties{type:"line|bar|pie", data{labels[], datasets[{label,data[]}]}, insights[]}
- **cardWidget**: properties{subject,summary,timestamp,badgeText,variant:"error|warning|info",link{text,action}}
- **messageListWidget**: properties{subtitle, items[{title,subtitle,summary,badgeText,priority:"alert|warning|medium",timestamp,action,image}]}
- **changeListWidget**: properties{displayType:"percentage|raw|currency", items[{title,currentValue,previousValue}], messages[]} (auto-calcs % change)
- **multiRecordWidget**: properties{cols[], rows[{cells[ string | {type:"badge",text,priority:"alert|warning|medium|success"} ], action{text,command}}]}
- **recordWidget**: properties{id, readOnly, fields[{id,type:"text|textarea|number|date|select",label,value,options[{value,label}]}]} (readOnly=false → editable form)
- **sankeyWidget**: properties{nodes[{id,name}], edges[{source,target,value}]}
Selection (DD 45): single alert→card; multiple→messageList; KPI→changeList; table→multiRecord; trend→chart; form→record; flow→sankey.

### 4.2 oraInsight — Actionable Insights (DD 49-51), emitted on `InitActions`
```json
{"title":"Review Critical Alert","shortDescription":"CPU exceeded 80%.","actionText":"Investigate",
 "followUpCommand":"ora.Invoke(\"openMonitoring\", \"cpu-alert-001\")","priority":true}
```
followUpCommand must be one of: `ora.Invoke("action","context?")` (run a predefined Application Action), `ora.Agent(content)` (send to an agent/LLM), `ora.App.*` (UI/nav/state: `ora.App.update()`, `ora.App.navigate(...)`, `ora.App.launch("APPCODE",{...})`, `ora.App.setConfig({...})`). Actions are produced by AND handled by the same agent on invoke.
**Enable Actions = 3 points (DD 49):** (1) LLM node Output Tab → "Enable Actions for the Widgets"; (2) App Builder Agent Editor → "Include in Actions" + Initial Actions Prompt; (3) workflow `InitActions` branch emits `<oraInsight>`.

### 4.3 Action Editor step types (DD 52-54)
+ New Action → Code + Display Name + Description → Add Step. Steps: **Keep Action Available in UI**, **Navigate to App** (target app Code + payload), **Send Agent Command** (`ora.Agent('refresh')`), **Refresh Agents** (pick agents to re-render), **Switch App Context** (new context value, optional Reload App).

### 4.4 $OraAction flow (DD 55)
click → `ora.Invoke()` fires actionName+context → `$context.$app.$OraAction` populated → workflow branches on `InvokeAction` then `$OraAction.includes("name")`. `$OraAppContext` = launch-time scope (URL params / record), e.g. `ora.App.launch("COMPLIANCE",{recordId:"12345"})`.

### 4.5 oraComms (DD 61-66) — emitted on `InitCommunications`
Two flavors: **App-Defined** (Builder templates: PowerPoint/PDF/Email/Text, params filled at runtime) and **Agent-Generated** (runtime `<oraComms>`, Email/Text only, MUST include real content — no placeholders).
```json
{"type":"email","title":"Workshop Update","content":"Hi Laura,\n\n...","params":[{"id":"email","defaultValue":"Laura@example.com"},{"id":"subject","defaultValue":"..."}]}
```
Text adds `"targetAgent":"COMMS_SENDER"`. Lifecycle (DD 66): Suggest(`InitCommunications`) → Prefill(framework) → Fill(`FillParameters`) → Review(user) → Send(`SendCommunication`). Same agent, different prompts. Template Editor (DD 63): type, title, sections (each: allow-agent-fill toggle + prompt + presentation instructions); link template to a Communication (Title, Description, Applicable Agents, Template).

## 5. Per-agent prerequisites (DD 29/31/32/36)
1. **Published** (Draft agents not selectable).
2. **Output Tab → "Expose to Agentic Apps"** ON (else no hint routing).
3. **App Experience sub-tab**: Enable Actions/Communications for widgets; **Select Widgets** (Card/MessagesList/ChangeList/Chart/Record/MultiRecord/Sankey); Instructions Prompt.
4. **Switch on `$OraMessageHint`** with cases: InitDisplay, Summary, InitActions, InitCommunications, Query, InvokeAction, FillParameters (+AdditionalContent). Default → Query.
5. **Agent Editor** fields filled in App Builder.
Ask Oracle agent (DD 57-58): Query node must receive `$context.$system.$inputMessage` + `$context.$system.$chatHistory` (history NOT auto-passed → follow-ups break without it). Orchestrator routes by **agent description** — write specific descriptions.

## 6. Context parameters (DD 23-24)
`$context.$app.$OraMessageHint`, `.$OraAction`, `.$OraAppContext`, `.$OraUserContext`, `.$OraAttachments`; `$context.$system.$chatHistory`, `.$inputMessage`. `$OraUserContext` = {fullName,userId,userName,guId}.

## 7. Best practices (DD 72-74)
- **Init context first**, fetch shared data once, parallelize independent calls, route by hint, end each branch with a terminal node.
- One focused agent per panel/domain; strong differentiated descriptions; control app participation per agent; reusable logical objects (security once).
- **Separate recommendation from execution** (generate action → execute only after user approval) — the HITL gate. Structured action payloads (type/target/recordId/params/op). Test each branch independently; review traces.
- Orchestrator/supervisor is platform-handled — your lever is precise agent descriptions + Switch-on-hint + `$OraAction` branching.

## 8. Seeded app catalog
**HCM (7, OV 4/16):** Manager Concierge Workspace; Workforce Operations Command Center; Team Learning & Development Workspace for Managers; Hiring Workspace for Store Managers; Career Advancement Command Center; Team Talent Calibration & Review Workspace; My Help Workspace for Employees.
**ERP/SCM (11):** Design-to-Source / Product Readiness / Production Shift Operations / Sales Order Command Center / Batch Process Manufacturing / Logistics Execution Command Center / Maintenance Operations / Warehouse Operations / Cost Accounting Close / Sourcing Command Center / Collectors Workspace / Security Command Center.
**CX (4):** Cross-Sell Program / Contract Compliance / Sales Command Center.

## 9. Marketplace (BP 5-8): no separate app artifact to submit — submit the constituent agent JSONs + a setup guide that documents every Builder field; app is 26B+ only; agents must be hostname-free; Oracle re-imports agents and rebuilds the app shell manually. Design-fit (DD 70): cross-system synthesis, multi-step workflow, in-context actionability, judgment/prioritization, outcome (not just answer).
