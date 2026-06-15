---
name: oracle-hcm-agent-builder
description: "Build Oracle AI Agent Studio import JSON files for Oracle Fusion HCM. Use this skill whenever the user wants to create an AI agent, build an agent team, generate an agent JSON, design a multi-agent workflow, or create tools for Oracle AI Agent Studio. Also triggers on: 'build an HCM agent', 'create agent JSON', 'Agent Studio import', 'supervisor worker pattern', 'oracle agent', 'compass agent', 'document tool', 'business object tool', 'agent team', 'workflow agent', or any request involving Oracle Fusion AI agents. Use this even for modifying, fixing, or debugging existing agent JSON files."
---

# Oracle HCM Agent Builder

You are building import-ready JSON files for Oracle AI Agent Studio — the design-time environment inside Oracle Fusion Cloud where AI agent teams are created, tested, and deployed.

This skill encodes hard-won rules learned through trial and error. Following them exactly is the difference between a successful import and a cryptic Studio error.

## Portability & Fallbacks

This skill is self-contained:

- It requires NO other skill, NO "superpowers" framework, NO Skill tool, and no special plugin. If you are running in Codex, Copilot, Gemini, or a plain LLM context, just follow the prose directly — no harness-specific invocation needed.
- All knowledge here (the Studio JSON contract, the gotchas, the endpoint list, the sentinel map) is harness-independent. It applies regardless of which AI assistant or CLI you are using.
- File references in this skill are OPTIONAL helpers. If a referenced file (e.g. `references/verified_endpoints.md`) does not exist in the current project, ignore it and proceed using the specifications in this document — they are complete on their own.

## Project Configuration — replace per deployment

The values below are the ONLY things that should change between customers or pods. Set them once at the start of a build session and use them consistently throughout.

| Setting | Default / Placeholder | Notes |
|---|---|---|
| `Family` | `HCM` | Change to `ERP`, `CX`, `SCM`, etc. for other Fusion families. |
| `Product` | `GLOBAL_HUMAN_RESOURCES` | Change to the appropriate product code for the target module. |
| Document/UCM links | Use relative `/cs/idcplg?...` paths — no hostname needed; see the portability gotcha. | Never hardcode a pod hostname; there is no runtime variable for it. |
| Reference JSON (optional) | _(none — leave blank if not available)_ | If your project has a known-good exemplar agent JSON, name it here; otherwise rely entirely on the JSON specs in this skill. |

These settings drive everything else. Do not hardcode pod hostnames or product codes anywhere in the build.

## When to Use This Skill

Use this whenever someone asks you to:
- Create a new AI agent or agent team for Oracle HCM
- Generate an import JSON for AI Agent Studio
- Fix or debug an existing agent JSON that fails to import
- Add tools (Business Object, Document, External REST) to an agent
- Design a multi-agent workflow (supervisor + workers)
- Build tools that call HCM REST APIs or use Document Tool RAG

## Architecture Patterns

Oracle AI Agent Studio supports three agent team types. Choose based on the use case:

### Hierarchical (Group) — Supervisor + Workers
**Use when:** Multiple specialized agents collaborate on a task. The supervisor routes requests to workers and synthesizes results. This is the most common pattern for complex workflows.

- `Architecture`: must be `"group"`
- One agent has `AgentType: "SUPERVISOR"`, all others are `"WORKER"`
- The supervisor's prompt controls sequencing — workers don't talk to each other directly
- All `agentMappings` edges originate from the supervisor
- **Limitation:** `group` architecture cannot reference cross-artifact REUSABLE agents. Workers must be inline in `agents[]`. Use `data_pipeline` if you need cross-artifact reuse.

### Workflow (data_pipeline) — Node-based pipeline
**Use when:** The process is a fixed sequence of steps (like a pipeline). Nodes execute in order with branching logic. Also the required architecture for cross-artifact REUSABLE agent references.

- `Architecture`: `"data_pipeline"` at the envelope root
- Uses `dataPipeline` with `pipelineNodes` instead of agents
- Supports node types: `START`, `END`, `LLM`, `CODE`, `SWITCH`, `LOOP`, `SET_FIELDS`, `BO_FUNCTION`, `REST`, `EMAIL`, `RAG_DOCUMENT_TOOL`, `VECTOR_DB_WRITER`, `VECTOR_DB_READER`, `AGENT`
- `agents: []` and `agentMappings: []` must be EMPTY at envelope root
- REUSABLE module agents are declared in `dependencies.agentNodes[]`
- Good for: automated processing pipelines, multi-step approval chains, multi-module orchestration

### Single Agent
**Use when:** A simple agent with tools that handles requests directly, no delegation needed.

- `Architecture`: `"single_agent"` at the envelope root
- One agent with its tools
- Set `ReusableFlag: true` on the inner agent to make it reusable by `data_pipeline` orchestrators
- Simplest pattern, good for FAQ bots and simple lookup agents

### Cross-artifact reuse pattern
Module workers = `single_agent` architecture with `ReusableFlag: true`. Orchestrators = `data_pipeline` referencing them as REUSABLE in `dependencies.agentNodes[]`. `group` architecture **cannot** do cross-artifact REUSABLE refs — it forces workers inline.

## JSON Structure — Complete Reference

The import JSON has four levels: Workflow → Agents → Tools → Agent Mappings. Every field listed below matters — missing or wrong fields cause silent import failures.

### Level 1: Workflow (root object)

```json
{
  "Specification": {
    "dataPipeline": {
      "pipelineNodes": [],
      "variables": [],
      "errorHandlers": []
    },
    "jsonSchemaName": "Workflow.spec",
    "jsonSchemaVersion": "1",
    "inputs": [],
    "outputSpecification": " ",
    "agentsValueMappings": [],
    "humanApprovalFlag": false,
    "triggers": [
      {
        "type": "REST",
        "inputs": []
      }
    ],
    "deletedTriggerTypes": [],
    "chatExpTPFileUploadEnabledFlag": false,
    "chatExpThirdPartyDetails": []
  },
  "WorkflowCode": "YOUR_WORKFLOW_CODE",
  "Name": "Your Agent Team Name",
  "Description": "What this agent team does",
  "Family": "HCM",
  "Product": "GLOBAL_HUMAN_RESOURCES",
  "HiddenFlag": false,
  "AiAppsCompatibleFlag": null,
  "UseCaseId": "NA",
  "SourceWorkflowId": null,
  "AccessModifier": "public",
  "StartQuestionOne": "Example question 1",
  "StartQuestionTwo": "Example question 2",
  "StartQuestionThree": "Example question 3",
  "FollowUpPromptEnabledFlag": false,
  "FollowUpPrompt": "Based on the following conversion: $param.system_context.chat_history generate $param.system_context.number_of_follow_up_questions\nfollow up questions only while strictly adhering to the JSON Schema format mentioned below. Remove the ```json markdown from the output.\nHere is the JSON Schema format the output should adhere to:\n[{\"question\": \"<put first question generated here>\"}, {\"question\": \"<put second question generated here>\"}, {\"question\": \"<put third question generated here>\"}",
  "Architecture": "group",
  "MaximumInteractions": 15,
  "StartAgentId": null,
  "agents": [],
  "agentMappings": [],
  "partnerMetadata": {
    "Name": "Your Agent Team Name",
    "Description": "Brief description"
  }
}
```

**Critical workflow rules:**
- `outputSpecification`: must be `" "` (single space, NOT empty string, NOT a JSON schema) — empty string causes import failure; a schema short-circuits the terminal LLM and it never runs
- `StartAgentId`: must be `null` — Studio finds the supervisor by AgentType
- `AiAppsCompatibleFlag`: include as `null`
- `partnerMetadata`: only two fields allowed: `Name` and `Description`
- Do NOT include: `policyIds`, `costSavings`, `timeSavings`, `modelConfiguration`, `waitFlag` in Specification
- The `FollowUpPrompt` string above is Oracle's standard template — use it verbatim
- `WorkflowCode`: must be FRESH and never-previously-used for any content change — Studio caches the decomposed pipeline by `WorkflowCode`; reusing a code keeps stale pipeline even after delete-to-zero (see Deploy/Publish gotchas)

### Level 2: Agents (inside `agents` array)

```json
{
  "Specification": {
    "jsonSchemaName": "Agent.spec",
    "jsonSchemaVersion": "1",
    "agentRole": "Supervisor",
    "customFlag": true,
    "sourceObjectCode": "YOUR_AGENT_CODE",
    "inputs": [],
    "outputSpecification": " ",
    "summarizationMode": "Custom",
    "summarizationPrompt": "How to summarize this agent's output for the supervisor"
  },
  "AgentCode": "YOUR_AGENT_CODE",
  "Name": "Agent Display Name",
  "Description": "What this agent does",
  "Family": "HCM",
  "Product": "GLOBAL_HUMAN_RESOURCES",
  "ModuleId": null,
  "Namespace": "HCM.GLOBAL_HUMAN_RESOURCES",
  "SeededFlag": false,
  "PromptCode": null,
  "Version": 1,
  "MaximumInteractions": 15,
  "AgentType": "SUPERVISOR",
  "Prompt": "The agent's system prompt goes here",
  "ModelConfigId": null,
  "ReusableFlag": false,
  "tools": [],
  "topics": []
}
```

**Critical agent rules:**
- `AgentType`: only `"SUPERVISOR"` or `"WORKER"` — no other values
- `agentRole`: `"Supervisor"` or `"Worker"` (capitalized differently from AgentType)
- `sourceObjectCode`: must match `AgentCode` exactly
- `outputSpecification`: must be `" "` (single space)
- `ModelConfigId`: at agent level, NOT inside Specification — use `null` if unknown
- `ReusableFlag`: at agent level, NOT inside Specification
- `PromptCode`: required on every agent, even as `null`
- `ModuleId`: include as `null` for custom agents
- `topics`: use `[]` (empty array) — **required even if empty**; missing it causes a `fileUpload TypeError` crash in the Studio UI importer
- `Specification` allowed keys: `jsonSchemaName`, `jsonSchemaVersion`, `agentRole`, `customFlag`, `sourceObjectCode`, `inputs`, `outputSpecification`, `summarizationMode`, `summarizationPrompt`
- A worker agent with zero tools is valid (pure LLM reasoning agent)

### Level 3a: Business Object Tools (REST API)

```json
{
  "Specification": {"jsonSchemaName": "Tool.spec", "jsonSchemaVersion": "1"},
  "RestTool": {
    "ObjectProperties": {
      "jsonSchemaName": "SupportedBusinessObject.objectProperties",
      "jsonSchemaVersion": 1,
      "tools": [
        {
          "headers": {"REST-Framework-Version": "9"},
          "bodyTemplate": "",
          "description": "What this function does",
          "id": 0,
          "isNew": false,
          "name": "function_name_v1",
          "operationType": "GET",
          "parameterDefinitions": [
            {
              "dataType": "string",
              "description": "Parameter description with format and valid values",
              "id": 1,
              "isToken": true,
              "name": "parameterName"
            }
          ],
          "resourcePath": "/hcmRestApi/resources/11.13.18.05/endpoint?q={filterQuery}&onlyData=true&limit={recordLimit}&offset={offsetValue}",
          "resourceType": "ADF_BC_FIXED_QUERY",
          "sampleQueries": [],
          "type": "BUS_OBJECT"
        }
      ]
    },
    "ObjectCode": "YOUR_TOOL_CODE",
    "Family": "HCM",
    "Product": "GLOBAL_HUMAN_RESOURCES",
    "ObjectSource": "ADF_BC",
    "Category": null,
    "RestResourcePath": "/hcmRestApi/resources/11.13.18.05/endpoint",
    "RestResourceIdentifier": "-1",
    "RestSupportedOperations": "{}",
    "ParentSupptObjectId": null,
    "MandatoryFlag": false,
    "SeededFlag": false,
    "ObjectDisplayName": "Tool Display Name",
    "Name": "Tool Display Name",
    "Description": "What this tool does"
  },
  "ToolCode": "YOUR_TOOL_CODE",
  "Type": "BUS_OBJECT",
  "Name": "Tool Display Name",
  "Description": "What this tool does",
  "Family": "HCM",
  "Product": "GLOBAL_HUMAN_RESOURCES",
  "SeededFlag": false,
  "HiddenFlag": false,
  "ModuleId": null,
  "Namespace": "HCM.GLOBAL_HUMAN_RESOURCES",
  "Version": 1,
  "UserInputRequiredFlag": false,
  "UserInputMsg": null
}
```

**Critical BUS_OBJECT rules:**
- `Specification` at the tool level: must be `{"jsonSchemaName":"Tool.spec","jsonSchemaVersion":"1"}` — NOT `{}`. Using an empty object is wrong and causes import issues. (Document Tool `Specification` stays `{}` — only BUS_OBJECT uses the schema value.)
- `headers` on every inner tool: MUST be `{"REST-Framework-Version":"9"}` — the `headers` key must exist (the import parser requires it) AND its value must carry the version (the ADF `q=<Key> IS NOT NULL` sentinel used by every GET tool is only valid in REST framework v2+; empty `{}` falls back to v1 and returns HTTP 400 on IS-NOT-NULL and LOV queries). A 404 after the header is correct indicates a privilege gap, not a query problem.
- `ObjectSource`: must be `"ADF_BC"` (not `"CUSTOM"`)
- `RestResourcePath`: must use versioned format like `/hcmRestApi/resources/11.13.18.05/...` — NEVER use `/latest/`
- `RestSupportedOperations`: must be string `"{}"` (stringified empty object, not an actual object)
- `RestResourceIdentifier`: must be string `"-1"`
- Inner `resourceType`: must be `"ADF_BC_FIXED_QUERY"`
- Inner `type`: must be `"BUS_OBJECT"`
- Inner `resourcePath`: must also use versioned format (no `/latest/`)
- Inner function `name`: must be suffixed with `_v<N>` on every new version — Studio dedups by inner function name; same name keeps the OLD definition
- `operationType`: `"GET"` for reads, `"POST"` for creates/actions
- Parameter `isToken: true` means the LLM fills it dynamically at runtime
- `bodyTemplate`: use `""` for GET, use a JSON template string for POST

### Level 3b: Document Tools (RAG)

```json
{
  "Specification": {},
  "ToolCode": "YOUR_DOC_TOOL_CODE",
  "Type": "DOCUMENT",
  "Name": "Document Tool Name",
  "Description": "What documents are indexed here and when to search them",
  "Family": "HCM",
  "Product": "GLOBAL_HUMAN_RESOURCES",
  "SeededFlag": false,
  "HiddenFlag": false,
  "ModuleId": null,
  "Namespace": "HCM.GLOBAL_HUMAN_RESOURCES",
  "Version": 1,
  "UserInputRequiredFlag": false,
  "UserInputMsg": null
}
```

**Critical Document Tool rules:**
- `Type`: must be `"DOCUMENT"` — not `"BUS_OBJECT"`, not `"RETRIEVAL"`
- Do NOT include a `RestTool` block — Document tools have no REST endpoint
- `Specification`: use `{}` (empty object) — this is correct for Document tools (different from BUS_OBJECT)
- Post-import: upload files to the Document Tool in Studio UI, set status to "Ready to Publish", then run `Process Agent Documents` ESS job
- Supported file types: PDF, DOCX, PPTX, XLSX, HTML, Markdown, TXT, CSV, JSON, XML, PNG, JPEG, ZIP
- Limits: 500 files per document group, 300 pages per attachment, 2GB total size
- Documents must be re-uploaded when migrating between environments

### Level 4: Agent Mappings

```json
"agentMappings": [
  {
    "EdgeOrder": 0,
    "AgentCode": "SUPERVISOR_CODE",
    "AgentTargetCode": "SUPERVISOR_CODE"
  },
  {
    "EdgeOrder": 0,
    "AgentCode": "SUPERVISOR_CODE",
    "AgentTargetCode": "WORKER_1_CODE"
  },
  {
    "EdgeOrder": 1,
    "AgentCode": "SUPERVISOR_CODE",
    "AgentTargetCode": "WORKER_2_CODE"
  }
]
```

**Critical mapping rules:**
- Only 3 fields per edge: `EdgeOrder`, `AgentCode`, `AgentTargetCode`
- Do NOT include: `SourceTopic`, `TargetTopic`, `Route`, `Trigger`, `PayloadContract`, `HandOffMode`, `FailureRoute`, `MappingCode`
- ALL edges must originate from the SUPERVISOR agent only (no worker-to-worker edges)
- Must include a self-referencing supervisor mapping: supervisor → supervisor (EdgeOrder 0)
- Worker sequencing is controlled by the supervisor's prompt, not by mapping edges
- EdgeOrder determines the visual layout in Studio, not execution order

## Verified HCM REST API Endpoints

Only use endpoints that actually exist. Many HCM config objects do NOT have REST APIs. If present at project root, read `references/verified_endpoints.md` for the full list with paths.

### Full CRUD Resources (14 confirmed)
positions, jobs, grades, locations, organizations, gradeLadders, gradeRates, jobFamilies, goalPlans, recruitingCEJobRequisitions, valueSets, commonLookups, setupTasks, DFF Contexts

### LOV-Only Endpoints (read-only lookups)
departmentsLov, personTypesLOV, actionsLOV, actionReasonsLOV, payrollDefinitionsLOV, payrollBalanceDefinitionsLOV, salaryBasisLov, eligibilityProfilesLOV, benefitPlansLOV, absenceTypesLOV, successionPlansLOV, hcmBusinessUnitsLOV, legalEmployersLov, hcmTreesLOV, assignmentStatusTypesLov, performanceTemplateDocumentNamesLOV

### No REST API — Use Document Tool RAG
departments (standalone), elements, elementLinks, payrolls, fastFormulas, compensationPlans, accrualPlans, performanceTemplates, candidateSelectionProcesses, learningItems, extensibleFlexfields, approvalRules, workforceStructures

NEVER create BUS_OBJECT tools pointing to endpoints that don't exist.

## Standard Tool Parameters

Most GET tools should include these three parameters:

```json
"parameterDefinitions": [
  {
    "dataType": "string",
    "description": "Oracle ADF q= filter expression. SINGLE-EQUALITY ONLY: field='value'. NO compound AND/OR (Akamai WAF blocks). NO LIKE %...%. NEVER pass '1=1' (Oracle rejects with HTTP 400). For ALL ROWS: pass '<PrimaryKey> IS NOT NULL' (e.g., 'JobId IS NOT NULL' for jobs). Examples of filtered: ActiveStatus='A', JobCode='SE_001'.",
    "id": 1, "isToken": true, "name": "filterQuery"
  },
  {
    "dataType": "string",
    "description": "Maximum records to return. '10' for quick lookups, '500' standard, '2000' full exports. Default: 500. Pass '1' when only a count is needed (use with totalResults=true in the resourcePath).",
    "id": 2, "isToken": true, "name": "recordLimit"
  },
  {
    "dataType": "string",
    "description": "Records to skip for pagination. '0' first page, '500' second page. Default: 0.",
    "id": 3, "isToken": true, "name": "offsetValue"
  }
]
```

Standard resource path pattern:
```
/hcmRestApi/resources/11.13.18.05/{resource}?q={filterQuery}&onlyData=true&limit={recordLimit}&offset={offsetValue}
```

### CRITICAL — the no-filter sentinel

Oracle ADF rejects empty `q=` (HTTP 400) AND rejects column-shaped sentinels like `q=1=1` (HTTP 400 — "Left side of search operator must be the name of a searchable attribute"). Studio's resourcePath template substitutes parameters unconditionally — there is NO "omit if empty" mechanism — so `q={filterQuery}` always emits `q=<something>`.

**The only safe all-rows sentinel is `<PrimaryKey> IS NOT NULL`.** Per-resource map (verified):

```
jobs                              → JobId IS NOT NULL
positions                         → PositionId IS NOT NULL
grades                            → GradeId IS NOT NULL
gradeLadders                      → GradeLadderId IS NOT NULL
gradeRates                        → RateId IS NOT NULL
locations                         → LocationId IS NOT NULL
organizations / departments       → OrganizationId IS NOT NULL
departmentsLov / legalEmployersLov → OrganizationId IS NOT NULL
jobFamilies                       → JobFamilyId IS NOT NULL
workers / publicWorkers / payslips → PersonId IS NOT NULL
hcmBusinessUnitsLOV               → BusinessUnitId IS NOT NULL
actionsLOV                        → ActionId IS NOT NULL
actionReasonsLOV                  → ActionReasonId IS NOT NULL
hcmTreesLOV                       → TreeCode IS NOT NULL
personTypesLOV                    → PersonTypeId IS NOT NULL
goalPlans                         → GoalPlanId IS NOT NULL
developmentGoals / performanceGoals → GoalId IS NOT NULL
successionPlansLOV / benefitPlansLOV → PlanId IS NOT NULL
performanceTemplateDocumentNamesLOV → TmplPeriodId IS NOT NULL
learningCommunities               → learningItemId IS NOT NULL
salaryBases / salaryBasisLov      → SalaryBasisId IS NOT NULL
totalCompensationStatements       → StatementName IS NOT NULL
payrollDefinitionsLOV             → PayrollId IS NOT NULL
payrollRelationships              → PayrollRelationshipId IS NOT NULL
payrollBalanceDefinitionsLOV      → BalanceTypeId IS NOT NULL
payrollFlowInstances / flowInstances / processResults → FlowInstanceId IS NOT NULL
eligibilityProfilesLOV            → EligyPrflId IS NOT NULL
absences                          → absenceCaseId IS NOT NULL
absenceTypesLOV                   → AbsenceTypeId IS NOT NULL
fastFormulas                      → FormulaId IS NOT NULL
recruitingCEJobRequisitions       → Id IS NOT NULL
documentRecords                   → DocumentsOfRecordId IS NOT NULL
```

The build script's tool factory should accept a `key_field` arg per resource and embed `'<key_field> IS NOT NULL'` in the tool description's "for all rows…" hint AND in the sample query so the LLM has a working example.

### Counts — ADF page cap and totalResults

**ADF REST caps every response page at 500 rows.** `items.length` and the `count` field reflect page size only — they cap at 500 and are NOT the true total.

For an accurate count: append `&totalResults=true` to the resource path and read `response.totalResults`. Combine with `limit=1` to keep the payload tiny (you only want the count, not the rows):

```
/hcmRestApi/resources/11.13.18.05/workers?q=PersonId IS NOT NULL&totalResults=true&limit=1
→ items: [1 row], count: 1, totalResults: 5641, hasMore: true
```

Teach this in tool descriptions and prompts: for "how many X" questions, pass `recordLimit='1'`, read `response.totalResults`, and cite the exact integer — never report `items.length`.

## Prompt Writing Best Practices

### Supervisor Prompt Structure
1. Identity and role — what this agent team does
2. Data sources — Document Tool RAG (primary) and REST APIs (fallback)
3. Example user questions — concrete examples of how users will phrase requests
4. Routing logic — which worker handles what, in what sequence
5. Follow-up handling — conversational refinements
6. Response guidelines — never expose internals, use clean formatting
7. Scope — what modules and objects are covered

### Anti-Hallucination Grounding
Include in every agent that retrieves or presents data:
- "NEVER generate, assume, or fabricate data not returned by your tools"
- "If a tool returns no data, report that explicitly — do NOT fill gaps"
- "Every item in your output MUST trace back to a specific tool response"

### Document Tool Fallback Pattern
Include in every agent that uses Document Tool:
- "Try Document Tool RAG FIRST"
- "If it returns ANY error, IMMEDIATELY fall back to REST API tools"
- "Do NOT stop or report Document Tool errors — silently switch"
- "The Document Tool may not be configured yet. This is EXPECTED."

## Building Process

### 1. Clarify Purpose
What problem? Who uses it? What data? What output format?

### 2. Design Agent Team
Choose architecture. Define roles. Map data flow. Identify tools per agent.

### 3. Select Tools
Check verified endpoint list. Plan Document Tool RAG for objects without REST APIs. Pure LLM workers need zero tools. Retrieval/UCM agents need their OWN dedicated few-tool agent (see Portability/Reuse gotchas).

### 4. Generate JSON
Use exact field structures from this skill. Follow every critical rule.

### 5. Validate

Run through the checklist:
- [ ] `outputSpecification` is `" "` (single space) everywhere — NOT empty string, NOT a JSON schema
- [ ] `StartAgentId` is `null`
- [ ] `Architecture` is `"group"` for supervisor+worker (or `"data_pipeline"` for workflow)
- [ ] Every agent has `PromptCode`, `ModelConfigId`, `ModuleId` (even as null)
- [ ] Every agent has `topics: []` (empty array — required to avoid fileUpload TypeError)
- [ ] REST paths use versioned format (no `/latest/`)
- [ ] `RestSupportedOperations` is string `"{}"`
- [ ] `RestResourceIdentifier` is string `"-1"`
- [ ] `ObjectSource` is `"ADF_BC"`
- [ ] BUS_OBJECT tool `Specification` is `{"jsonSchemaName":"Tool.spec","jsonSchemaVersion":"1"}`
- [ ] Every inner BUS_OBJECT tool has `"headers":{"REST-Framework-Version":"9"}`
- [ ] All inner function `name` values are suffixed with `_v<N>` (unique per version)
- [ ] `WorkflowCode` is fresh and never-previously-used (content change = new code)
- [ ] Agent prompts contain zero ASCII `<` or `>` characters (use unicode `→`, `[brackets]`) — WAF check
- [ ] CODE nodes: no dense raw HTML/CSS — base64-encode if needed — WAF check
- [ ] All mapping edges from supervisor only
- [ ] Self-referencing supervisor mapping exists
- [ ] Document tools: `Type: "DOCUMENT"`, no `RestTool` block, `Specification: {}`
- [ ] `partnerMetadata` has only `Name` and `Description`
- [ ] Agent codes in mappings match actual agent codes
- [ ] All REST endpoints are verified (exist in the list above)
- [ ] data_pipeline terminal node is an LLM (not CODE) — a CODE terminal leaks the raw `{result,console,timedOut}` wrapper to chat
- [ ] No hostname literals anywhere in prompts, tool descriptions, or resource paths

## Platform Constraints

- **Text-only agents**: Agents cannot render clickable links, images, or files inline. For document downloads, emit a relative `/cs/idcplg?IdcService=GET_FILE&dID=<dID>&allowInterrupt=1` path that the user's signed-in browser resolves — describe the anchor in prose in the prompt; do NOT put a literal `<a` tag in the prompt (WAF).
- **No UCM direct access**: No Business Object for UCM GET/POST. The `erpintegrations/getDocumentForDocumentId` API works only for small files.
- **Document Tool upload is manual**: No REST API to programmatically upload to Document Tool. Upload via Studio UI, set "Ready to Publish", run `Process Agent Documents` ESS job.
- **Runtime File Processor**: The `MultiFileProcessor` tool lets agents process files uploaded in chat (up to 5 files, 50MB combined). It's a singleton — cannot be duplicated.
- **Agents run with user's Fusion token**: Data security stays intact — agents see only what the logged-in user can see in the UI.
- **Custom AI Agent Subscription required in production**: Custom agents (new agent + new integrations + new actions) require a paid subscription on the target pod. Non-production pods need an entitlement flag enabled. Symptom: HTTP 403 from Akamai CDN on `/hcmAgentsV2`, error code 27014 (`SupptObjectId`).
- **GPT-4.1-mini default in Studio**: The default agent model in Studio is `oci-agent/openai.gpt-4.1-mini-2025-04-14`. It tool-calls reliably but hallucinates aggressively when given negation lists or asked to slice missing data.
- **No pod hostname at runtime**: There is no system context variable (e.g. `$context.$system.$hostname`) that exposes the pod URL. The only portable approach for document links is a relative path (see Portability gotcha).

---

## Studio Build, Deploy & Runtime Gotchas

All of the following are verified by live debugging. Each rule = what happens + why + concrete fix.

---

### IMPORT

#### Gotcha: fileUpload TypeError on import (silent crash, no row added)

**Symptom:** Studio swallows the file silently — no row appears, no error toast, just a crash in `base-rest-config-bundle.js`: `TypeError: Cannot read properties of undefined (reading 'length')`. The error is in the Redwood UI's parse pass before any Oracle backend validation.

**Three root causes** (any one is enough to trigger the crash):
1. `agents[N].topics` is missing — must exist as `[]` (empty array).
2. `Specification.dataPipeline.errorHandlers[0]` has the wrong shape — must be the EMAIL handler:
   ```json
   {"type":"EMAIL","inputs":[
     {"name":"toList","type":"string","value":""},
     {"name":"ccList","type":"string","value":""},
     {"name":"subject","type":"string","value":""},
     {"name":"body","type":"string","value":""}
   ]}
   ```
   NOT a freeform `{"errorType":"...","errorMessage":"..."}` shape.
3. `agents[N].tools[i].RestTool.ObjectProperties.tools[j].headers` is missing — must exist on every BUS_OBJECT inner tool (even `{}` passes the import parser, but use `{"REST-Framework-Version":"9"}` for runtime correctness).

**Structural diff against a gold reference** before importing:
- `single_agent` architecture: diff against `PERSON_LEGISLATIVE_INFORMATION_SD.json`
- `group` architecture: diff against `WORKER_CONCIERGE_AM.json`

---

### WAF (Akamai/Kona)

The Akamai WAF blocks requests at the edge BEFORE Oracle auth. Studio mistranslates 403 responses as "We can't exit the sandbox right now" — this is a WAF block, not a sandbox issue.

#### Gotcha: Angle brackets in agent Prompt → 403 on agent POST

**Why:** ASCII `<` and `>` in the agent Prompt are scored by Akamai Kona as XSS tags (cumulative, inspects ~first 8KB of body). Three `<` plus 49 `>` (from `<Object>` placeholders and `->` routing arrows) can tip the score past the block threshold. Content is the trigger, not size — a 15KB benign body returns 400 (passes).

**Fix:**
1. Zero ASCII `<` and `>` in any agent Prompt. Use unicode `→` for arrows; `[brackets]` for placeholders.
2. Also lower the cumulative score: no inline `style="..."`, no `<table>`/`<td>` tag names, no `?IdcService=GET_FILE&dID=` query strings, no backticks, no SQL literals (`IS NOT NULL`, `='A'`, `GROUP BY`, `1=1`, `LIKE %`). Move SQL syntax to tool `parameterDefinitions` which POST separately and pass.
3. Describe HTML output shapes in **prose** in the prompt (e.g., "an anchor element, tag name a, with an inline style attribute setting display to inline-block…") — the model emits the real HTML at runtime, which is NOT WAF-inspected.

**Probe before every PATCH/POST:** `POST /hcmRestApi/resources/latest/hcmAgentsV2` with `{"AgentCode":"ZTEST","Name":"t","Prompt":<your-text>}` and basic auth — `403` = WAF will block; `400` = safe to deploy.

#### Gotcha: Dense HTML/CSS in a CODE node → workflow stuck DRAFT on publish

**Why:** The workflow publish POST carries the full `dataPipeline` Specification. Dense HTML + inline CSS (e.g. `style="font-family:-apple-system,...;color:#111827;"`) in a CODE node's `sourceCode` trips the WAF — the publish POST 403s and the workflow stays DRAFT. Plain-text CODE nodes publish fine.

**Fix:** Base64-encode the HTML string inside the CODE node and decode at runtime with an inline loop (no function declarations — the engine forbids `function foo(){}` but allows inline loops and arrow expressions). Keep the `=` padding in the base64 string — stripping it causes incorrect decoding at the padding boundary. Working inline decoder pattern:
```
var T="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
var i=0,out="";
while(i<B.length){
  var c1=T.indexOf(B.charAt(i++)),c2=T.indexOf(B.charAt(i++)),
      c3=T.indexOf(B.charAt(i++)),c4=T.indexOf(B.charAt(i++));
  out+=String.fromCharCode((c1<<2)|(c2>>4));
  if(c3>=0)out+=String.fromCharCode(((c2&15)<<4)|(c3>>2));
  if(c4>=0)out+=String.fromCharCode(((c3&3)<<6)|c4);
}
```

---

### CODE NODE

#### Gotcha: Regex sandbox quirk — `\b` and optional-hyphen tokens misbehave

**Why:** `\b` word boundaries and optional-hyphen patterns like `w-?2` pass in Node.js but fail in Studio's CODE sandbox — the branch falls through to the default case. Example: `/\bw-?2\b/.test("show me my w-2")` returns `true` in Node but routes to the default branch in-sandbox.

**Fix:** Use plain alternation instead of `\b`: `(w2|w-2|w 2|w_2)` not `\bw-?2\b`. Use char classes: `w[- ]?2`. Always include a `\b`-free co-occurring backup keyword (e.g. `tax (form|withholding)`) so the branch fires even if the token match fails. Prefer a 1-token LLM router over regex (see Routing gotchas) to sidestep the sandbox entirely.

#### Gotcha: CODE node limits (field-observed, undocumented)

- ~5s execution timeout
- ~100K-statement cap
- Undocumented body-size limit (empirically ~220KB) — test empirically for 413 or truncated output
- `function foo(){}` declarations are forbidden — use inline expressions, loops, and arrow callbacks
- Current-message variable: `$context.$system.$inputMessage`; chat history variable: `$context.$system.$chatHistory` (`$OraChatHistory` is deprecated)

---

### DATA_PIPELINE (workflow architecture)

#### Gotcha: Terminal node must be an LLM, not CODE

**Why:** A CODE node as the terminal step leaks its raw `{result, console, timedOut}` wrapper verbatim into the chat response. Oracle's own seeded examples always end with a terminal LLM node that echoes the CODE output via `{{$context.$nodes.<CODE>.$output.result}}`.

**Rule:** The last node before END must be an LLM node. Its job is to echo/render the upstream CODE or AGENT output — not to re-reason. Keep its prompt short and format-neutral.

#### Gotcha: `outputSpecification` must be a single space

`" "` (single space) is required. A JSON schema as `outputSpecification` makes Studio short-circuit at the last CODE node and the terminal LLM never runs. This was the single most common pipeline bug.

#### Gotcha: LLM node temperature must be numeric

A `temperature` input with a **string** value (e.g. `"0"` instead of `0`) silently disables the LLM node — it never executes. Use a numeric literal or omit the field entirely. The same applies to `seed` and `modelId` — all are supported for determinism but must be correctly typed.

#### Gotcha: Never chain AGENT→AGENT in a pipeline

Sequential `AGENT` → `AGENT` pipelines do NOT pass output context cleanly — the downstream agent re-answers `$inputMessage` from scratch with no tools. Always use `AGENT → LLM` where the LLM node consumes upstream output via `{{$context.$nodes.<NODECODE>.$output}}`.

---

### DEPLOY / PUBLISH / RE-IMPORT HYGIENE

#### Gotcha: Re-importing the same WorkflowCode keeps stale pipeline (root cause)

**Why:** Studio caches the decomposed pipeline by `WorkflowCode`. Even after delete-to-zero and node-id bumping, re-importing the same code keeps serving the STALE early decomposition — the new `sourceCode` never lands.

**Fix (both steps required):**
1. Use a FRESH, never-previously-used `WorkflowCode` for every content change.
2. Bump every pipeline-node `id` to a fresh unique series each build, and update `outcomes.success` cross-refs.
3. Delete the existing record (REST count must hit 0) BEFORE importing.

#### Gotcha: Debug chat caches per (workflow, question)

Re-asking the same question after a redeploy returns the OLD cached answer. Verify every redeploy with a **never-asked** or varied question to confirm the new content is live.

Published rows are hidden unless the "Published" filter chip is selected; fresh imports land as DRAFT.

#### Gotcha: Iterate prompt vs tools differently

- **Prompt-only change:** `PATCH /hcmAgentsV2/{AgentId}` with `{"Prompt": <clean>}` → 200. Fast loop, no reimport.
- **Tool change** (resourcePath, params, headers): re-import MERGES tools by inner function name and keeps the old definition. You MUST `DELETE` the tool record (and agent + envelope) first, then reimport fresh.
- **Publish a workflow:** `PATCH /hcmAgentWorkflowsV2/{WorkflowId}` with `{"Status":"PUBLISHED"}` → 200. Works even when the Studio UI Publish button fires no REST call.
- **Eval runs require a UI Publish**, not just the REST PATCH — the evaluator reads the published version.
- **Workflow metadata is decoupled from the compiled pipeline.** A `GET hcmAgentWorkflowsV2/{id}` returns ~30 metadata fields (Name, Status, WorkflowCode, etc.) but NOT the `Specification`/`dataPipeline` content. To fetch the pipeline, use `GET hcmAgentWorkflowsV2/{id}?fields=Specification`.

---

### ROUTING

#### Gotcha: Single-token LLM router is more reliable than regex CODE router

**Why:** A 1-token LLM router (emits ONE word: `HCM`, `ERP`, `CX`, `ANSWER`, `REFUSE`) + a downstream SWITCH node sidesteps the Studio CODE sandbox regex quirk entirely and handles intent + safety judgments that regex cannot.

**Rules:**
- The router must emit ONLY the bare token — no trailing text, forwarded message, or explanation. If the output contains anything beyond the token, the SWITCH exact-match fails and everything hits the default branch.
- The router should judge **intent + safety only** and **DEFAULT TO ANSWER** when uncertain. If it also judges domain membership, it over-refuses obscure abbreviated object names. Delegate object validity to a downstream grounding or validation step.
- `AGENT → LLM router output` is accessed as `{{$context.$nodes.ROUTE.$output}}` (not `.result` — that is CODE node syntax).

---

### RENDERING / UX

#### Gotcha: Chat renderer forces `p{display:inline}` — blocks collapse without explicit override

**Why:** The Studio chat rich-text renderer (`oj-sp-output-rich-text`) applies a non-`!important` rule setting every `<p>` to `display:inline`. Vertical `margin`, `padding`, and `border-bottom` on `<p>` are silently ignored on inline boxes — blocks run together even though the style attribute is present.

**Fix:** Put `display:block;` in every block element's inline style. A plain inline `display:block` overrides the rule. Example: `<p style="display:block;margin:0 0 14px;padding-bottom:12px;border-bottom:1px solid #e5e7eb">`. Applies to every `<p>`, `<ul>`, or container the terminal LLM echoes.

#### House render guidance (portable, palette-neutral)

For consistent, low-latency agent output:
- Ship 3 start questions (`StartQuestionOne`, `StartQuestionTwo`, `StartQuestionThree`) and a "You might also ask:" block of 3 follow-ups per response.
- Prefer **block cards** over heavy HTML tables. A styled HTML table (with `<colgroup>`, `<thead>`, `<td>` markup) roughly doubles echoed bytes versus equivalent text blocks → latency is proportional to echoed tokens, so tables roughly double latency.
- Each card: a block element with `display:block;margin:0 0 14px;padding-bottom:12px;border-bottom:1px solid #e5e7eb`. Use bold labels (`Risk:`, `Priority:`, `Why it matters:`).
- Palette: system font stack `-apple-system, BlinkMacSystemFont, Segoe UI, sans-serif`; a neutral heading color (choose the customer's brand color — do not hardcode a specific hex); grays `#374151`/`#4b5563`/`#6b7280` for secondary text; dividers `#e5e7eb`.
- Footer pattern: a Tip line, then "You might also ask:" + 3 follow-up items, then a muted closing line.
- Describe the HTML output shape in **PROSE** inside LLM-node prompts — never put literal angle-bracket tags in a prompt (WAF). The model emits the real HTML at runtime, which is not WAF-inspected.

---

### PORTABILITY / REUSE ARCHITECTURE

#### Gotcha: Studio strips `links[]` from tool responses — use relative paths for document links

**Why:** Direct curl to ADF REST endpoints returns a `links` array with the pod hostname in `links[0].href`. Studio's tool-call layer strips the `links` array before passing the response to the LLM — the model has no real href to slice from and will hallucinate a placeholder hostname (commonly `fscm.example.com` from training data). There is also no runtime system variable exposing the pod hostname.

**Rule:** For UCM/document download links, emit a **relative path**: `/cs/idcplg?IdcService=GET_FILE&dID=<dID>&allowInterrupt=1`. The user's signed-in browser resolves it against whatever pod they are on — portable across every pod and customer, and correct because OAuth/CloudGate blocks non-browser REST-to-UCM auth anyway.

Describe the anchor/button in prose in the prompt (tag name `a`, relative href, `target` `_blank`, inline style described as prose) — never a literal `<a href=...` tag in the prompt.

**Still true from the original finding:** tell the LLM WHAT relative path to use (positive constraint), not what to avoid — negation lists cause the LLM to hallucinate variants of the negated example.

#### Gotcha: Dedicated retrieval agent — Studio filters tools per query

**Why:** Studio dynamically filters the `tools[]` it passes to the LLM per query (semantic relevance selection). A "fetch the extract" query posed to an agent with 40 tools may surface zero retrieval tools — Studio omits them as irrelevant. The agent then calls the wrong tool or says "I don't have access."

**Fix:** Give retrieval/UCM tools their OWN dedicated agent with only ~4 tools (list, find, fetch, detail). With few tools they always surface. The orchestrator routes a `HCM_DOC` branch to this agent and the data branch (`HCM`) to the domain agent.

**Tool audit rule:** to verify an agent's full toolset, grep the build artifact JSON — NOT a single LLM trace. Studio filters tools per query so one trace shows only a subset. A trace showing 25 of 38 tools is normal; it does NOT mean the other tools are unwired.

---

### RUNTIME / SECURITY

#### Gotcha: Business-Object tools 401 after pod weekly refresh

**Why:** A P2T (Patch-to-Test) pod refresh does NOT carry over OPSS security data. The security tables backing agent-runtime auth become stale. The UI token-relay still returns 200 (data and grants are intact); only the agent's runtime token relay fails.

**Remediation checklist (run after every pod refresh):**
1. Setup and Maintenance → Manage Administrator Profile Values → set `ORA_ASE_SAS_INTEGRATION_ENABLED` = Yes (Site level).
2. Run Scheduled Processes in order: `Import Resource Application Security Data` → `Import User and Role Application Security Data` → `Synchronize Default Data Privileges for Custom Job Roles` → `Refresh Access Control Data`.
3. Confirm the runtime user has `ORA_HRC_HCM_AI_AGENT_MANAGEMENT_DUTY` and `ORA_DR_FAI_GENERATIVE_AI_AGENT_HCM_ADMINISTRATOR_DUTY`.
4. Clear cache, log out/in, re-test a simple tool call (e.g. `/jobs?limit=1`). If still 401 → Oracle SR (OWSM/OPSS wiring for the `fusion-ai` runtime service principal).

**Design consequence:** don't put a live-REST AGENT node on a critical path that must always answer. Consider routing release/readiness queries through a config-aware prompt (no live REST) so they remain reliable even when the security tables are stale.

---

### VERSIONED RE-IMPORTS

#### Gotcha 1: Studio dedups inner tools by the function `name` field

**Why:** The inner tool `name` (the LLM-facing function name, e.g. `list_recent_extracts`) is what Studio uses to dedup tools across imports. Bumping `ToolCode`, `ObjectCode`, `ObjectDisplayName`, and `AgentCode` is NOT enough — if two versions share inner function names, Studio keeps the OLD inner-tool definition (descriptions, parameter docs, sample queries, headers) and silently rewires the new agent to the existing tool.

**Symptom:** The workflow imports as the new version, the agent reports its AgentCode correctly, but the LLM's tool calls show the old version's description text and behavior.

**Rule:** Every new version MUST suffix every inner function `name` with `_v<N>` (e.g., `list_extracts_v8`, `find_ucm_file_v8`). This is the only field Studio actually uses for dedup.

**Diagnostic protocol** when re-import behavior is suspicious:
```
Prompt the agent:
  "List the exact names of every tool you can call.
   One per line. Just the tool/function names."

If the response shows un-suffixed names, the new version didn't fully take effect.
Cleanup: delete prior-version tools by ObjectCode in Studio → Tools tab,
then re-import.
```

#### Gotcha 3: Oracle ADF rejects q=1=1 and q= (empty) as no-filter sentinels

When building a multi-resource query agent, the temptation is to pass `'1=1'` for "all rows". Oracle ADF rejects it: `"Failed to build ViewCriteria from expression '1=1' — Left side of search operator '=' must be the name of a searchable attribute"`. Empty `q=` also returns 400.

The only working all-rows sentinel is `<PrimaryKey> IS NOT NULL`, which differs per resource. See the per-resource key-field map under **Standard Tool Parameters — CRITICAL** above. Bake the right field into each tool's description AND its sample query.

#### Gotcha 4: GPT-4.1-mini ignores rule-only formatting instructions; needs literal example

**Why it happens:** Telling the LLM "BLANK LINE between sections" is insufficient. GPT-4.1-mini under Oracle Studio (Custom summarizationMode) reliably produces a wall-of-text response when instructed via rules alone. LLMs imitate examples far more reliably than they follow rule statements.

**Rule:** Embed a literal character-for-character EXAMPLE in the prompt showing the exact output shape with visible blank lines, then state "MIRROR THIS EXACT FORMAT".

Concrete pattern that works:
```
### REPLY EXAMPLE — copy this character-for-character format

There are 247 active jobs. Top 5 by manager level:

| Job Name | Code | Level |
|---|---|---|
| ... | ... | ... |

You might also ask:

- Break this down by job family
- Show me positions for these jobs
- Compare with last month's headcount

NOTICE the blank lines. They are REQUIRED.
```

Pair the example with a **summarizationPrompt** that also embeds the literal `\n\n` between sections — the summarizer is the last layer that touches the user-facing text, and if it collapses blank lines, all worker effort is wasted.

#### Gotcha 5: Routing rules trigger on incidental nouns; must classify by primary verb

**Why it happens:** A supervisor prompt that routes on keyword lists (e.g., "if the word 'Extract' appears, send to the file-retrieval worker") misfires when the same noun appears incidentally. Example: `"List active jobs from the Work Structure Extract"` routes to file retrieval because "Extract" matched — but the user's primary intent was a data query.

**Rule:** In the supervisor's routing prompt, classify by the **primary VERB and OBJECT**, not by any single noun. File-retrieval workers should fire ONLY on explicit file-deliverable verbs ("download", "give me the file", "the BR100", "the doc"). Data verbs (`list, show, count, find, by, with, how many`) override any incidental mention of "extract" or "config" elsewhere in the query.

Example overrides to put in the supervisor prompt:
```
"List active jobs from the extract"      → DATA worker
"Show me departments from the config"    → DATA worker
"Get me the latest config doc"           → FILE retrieval worker
"Where can I download the BR100?"        → FILE retrieval worker
```

#### Gotcha 6: User-facing footer (Source/Records/Confidence) is friction, not trust

Adding a `Source: <endpoint> | Records: <n> | Filters: <list> | As-of: <date>` footer plus a `Confidence: high — <reason>` line per response produces clutter and degrades UX in live testing. If audit/observability is needed for a regulated tenant, route it through Studio's session traces instead — do not display in chat.

#### Gotcha 7: Trivia must be data-specific, not file-descriptive

A trivia line like *"This document consolidates Jobs, Positions, Grades, Departments, Locations, Business Units, Legal Entities, and Job Families in one extract"* is a description of the file's structure, not an insight derived from the current run's data. It adds no value.

Real trivia is data-derived: a Pareto skew (`73% of jobs are in 3 families`), an outlier (`only 1 location in Denmark`), a singleton (`one position with FTE > 1.0`), an oldest record (`oldest active job created 2018-03-12`), a recent change (`5 new locations added this month`). The prompt MUST include 3–5 GOOD examples and 3–5 BAD examples — the LLM imitates the pattern from the examples.

---

### ADVANCED: Vector DB self-seeding (data_pipeline only)

`data_pipeline` supports `VECTOR_DB_WRITER`, `VECTOR_DB_READER`, `LOOP`, and `SET_FIELDS` nodes for self-seeding RAG at runtime. Key gotchas:

- The operation field is `operation` (not `operationType`); values: `INSERT`, `OVERWRITE`, `UPSERT`, `DELETE`.
- Declare `IndexName` in `Specification.dataPipeline.variables` as `[{name:"IndexName",scope:"JOB",type:"string"}]` — if `variables:[]`, `{{$context.$variables.IndexName}}` resolves empty → writes/reads hit a null index → silent `[]`.
- Do NOT set a strict `outputSpecification` on the READER node — it filters real output fields (like `vectorSearchScore`) to `[]` if they aren't in the declared schema. Leave it permissive or omit it.
- LOOP node requires `metadata.loopType:"SEQUENTIAL"` — omitting it throws "missing required metadata loopType".
- CODE node output in a LOOP: if your CODE does `return [...]`, the array is at `{{$context.$nodes.<CODE>.$output.result}}`, NOT `.items`.
- Embedding lag: a read immediately after a write can return `[]`; records become searchable after a short delay.
- Vector writes are NOT permission-aware — never store PII in a vector index.
- Limit writes to ~500 objects per workflow execution; batch larger corpora.

---

## REST Operations — Auth, Deploy, Publish, Eval & Trace

This section consolidates the operating model for scripting and automating agents over REST: which auth regime to use, how evals work end-to-end, how to harvest results, and how to debug. Everything here is grounded in verified observations from live pods.

---

### Auth model

There are **two distinct auth regimes**. Which one a call needs determines whether you can script it at all.

#### OAuth2 Bearer — required for agent invocation
`POST /api/fusion-ai/orchestrator/agent/v2/{code}/invokeAsync` requires an OAuth2 Bearer token. A session cookie returns 401; replaying `GET /fscmRestApi/tokenrelay?scopesfq=urn:opc:resource:fusion:...:fusion-ai/` also returns 401; harvesting the app's own token and replaying returns 401. Without IDCS confidential-client credentials this endpoint generally **cannot be driven from a plain script or a replayed session** — automated invocation via invokeAsync is not feasible without a registered OAuth client.

#### Session cookie — sufficient for Fusion REST and BOSS
All standard Fusion REST resources (`/hcmRestApi/...`, `/hcmAgentsV2`, `/hcmAgentWorkflowsV2`) and the BOSS data API (`/api/boss/data/objects/ora/commonFusionAI/agents/v1:3054/$en/evaluationRuns/{id}`) accept session-cookie auth. Use `credentials:'include'` (or the equivalent in your http client) — no Bearer token is required.

#### Why standalone curl/Python does not work — and what does
Fusion Cloud blocks HTTP Basic auth on REST in key surfaces (returns HTTP 500 or redirects to IDCS OAuth). The portable approach is an **in-browser fetch executed from inside a logged-in session**: HttpOnly session cookies auto-attach there but are invisible to `document.cookie`, and a terminal curl or Python script cannot replicate them. (environment-shaped — the mechanism, an in-browser fetch from a signed-in tab, is what matters; the specific tool to achieve it is up to your harness.)

#### Scripting gotcha when wrapping in-browser fetch
`page.evaluate` (or any equivalent that passes arguments to a browser script) passes exactly **one** argument object. If your fetch helper expects `(url, options)`, it must **destructure a single args array**: `async (a) => { const url = a[0], opt = a[1]; ... }` — otherwise `opt` (method, body, headers) is silently dropped and every call becomes a bare GET. BOSS endpoints also return 401 from the top-level welcome frame (treated as anonymous) but return 200 from inside the AI Studio page (`/fscmUI/redwood/human-resources/ai-studio`) where the BOSS token relay is established. (environment-shaped — applies to any in-browser fetch injection pattern.)

---

### Deploy & publish (brief recap — full rules above)

Prompt change: `PATCH /hcmRestApi/resources/11.13.18.05/hcmAgentsV2/{AgentId}` with `{"Prompt": <clean>}`. Tool change: delete + reimport (merge-by-inner-name means only a fresh import lands the new definition). Publish: `PATCH /hcmRestApi/resources/11.13.18.05/hcmAgentWorkflowsV2/{WorkflowId}` with `{"Status":"PUBLISHED"}`. Content change: fresh `WorkflowCode`. Pre-flight every agent PATCH/POST with the WAF probe before sending. See the **Deploy / Publish / Re-import Hygiene** and **WAF** gotchas above for the full rules.

---

### Running evals over REST

#### Eval runs require a UI Publish, not just the REST PATCH
`PATCH hcmAgentWorkflowsV2 {Status:PUBLISHED}` flips the workflow's Status field but does **not** build the agent-team runtime snapshot that the evaluation framework reads. A REST-PATCH-published workflow's eval run fails at `FetchEvaluationDataStep: Error fetching evaluation data`. Fix: open the agent team in Studio and click **Publish** (the "Publish custom agent team?" confirm). After UI-publish the eval run goes INPROGRESS instead of ERROR. REST PATCH remains correct for chat/runtime use; only evals need the UI publish.

#### Triggering a run via REST
The run-creation endpoint exists but requires a complete body: `{targetObjectState, status, metrics:[...]}` — an incomplete body returns 500. In practice, use the Studio UI: **Actions → Initiate Evaluation Run → Run** (set Version=Published in the drawer). Automation can click the footer Run button via a browser harness; the flow works even if a sandbox toast fires (see below).

#### Eval-Create POST is WAF-inspected
The `POST /api/boss/data/objects/ora/commonFusionAI/agents/v1:3054/$en/evaluations` is inspected by Akamai Kona like any other Studio POST. A single question row containing XSS token sequences — even with angle brackets replaced, e.g. `[script]alert(1)[/script]` — trips Akamai and returns **HTTP 403 from edge**. The Studio UI surfaces this as the toast pair "We can't exit the sandbox right now" + "We couldn't update your default view". This is a **WAF block, not a sandbox problem**. Fix the question payload; do not chase Configuration > Sandboxes.

#### "Can't exit the sandbox" toast is non-fatal for the Run
When clicking Run, the sandbox toast fires cosmetically (an MDS default-view write fails) but **the evaluation run still launches and executes in the background**. Do not abandon automation at the toast. Verify the run exists via the BOSS API, not the UI runs table — the table may not refresh if the ADF sandbox is stuck.

#### Stuck-sandbox → blind duplicate runs
A "couldn't update your default view" toast can mean the runs table **will not refresh** — the just-launched EVR row is invisible in the UI. If you click Run again thinking it failed, you create duplicate runs that contend for agent runtime (150 sequential questions already takes ~50 min for one run; duplicates make it worse). **SOP:** launch exactly one run per batch; confirm the EVR exists via `GET .../evaluationRuns?$filter=sessionIdentifier='EVR-...'` (BOSS API) before re-clicking. A fresh login on a wiped browser profile is the only reliable way to clear a stuck sandbox.

#### Eval set import format
The eval Create payload and the Studio "Add from File" import both accept **CSV** (header: `Question,Expected Response`). XLSX is rejected by the file picker. The CSV header row is imported as a junk question (question text = "Question") — delete it in the grid or strip it from the file. Keep question text WAF-safe: no `<`, `>`, `script`, `alert(` sequences.

---

### Harvesting eval results (BOSS API)

**Fetch a run record:**
```
GET /api/boss/data/objects/ora/commonFusionAI/agents/v1:3054/$en/evaluationRuns
    ?$filter=sessionIdentifier='EVR-...'
    &$fields=id,status,coverage,jobsCount,errorCount,medianCorrectness,specification
```
(environment-shaped — the `v1:3054` object id may differ per pod or release; confirm by inspecting the Studio network traffic on your pod.)

You **must** list `specification` in `$fields`. A default GET omits it.

**Parse results from `specification.evaluationResult[]`** — per-question keys:
`questionId`, `actualAnswer` (HTML), `score`, `answer_correctness_score`, `answer_correctness_explanation`, `latency`, `inputTokenCount`, `outputTokenCount`, `error`, `traceId`.

**Status value is `COMPLETE`** (not `"COMPLETED"` — code that waits on "COMPLETED" hangs forever).

**For per-question results from a specific run:** `POST .../evaluationRuns/{runId}/$query` with body `{fields:["specification"]}`, then `JSON.parse(specification).evaluationResult`.

**Aggregate/UI metrics** (correctness trend, monitoring aggregations, workflow filter dropdowns) come from the extraction queries the Studio UI uses: `/api/boss/extraction/queries/.../evaluationAggregations`, `monitoringMetricAggregations`, `distinctWorkflows`. The plain data API `GET evaluationRuns/{id}` returns sparse `{id,$id,$context}` unless `specification` is included in `$fields`.

**`distinctWorkflows` filter is prefix-match.** Type the leading token of the workflow name (e.g. the org prefix), not a mid-string word — `"Sentinel"` returns "No matches found"; `"NTT"` returns all NTT-prefixed workflows.

**Metrics to report for a submission package:** error rate, latency P50/P99, token usage (inputTokenCount/outputTokenCount per question), correctness vs reference answer.

---

### Trace & debug

- Every eval result row carries a `traceId` you can correlate to the Studio Monitoring log for that question.
- **ADMIN execution trace `nodes[].output` is always `null`** — even for SET_FIELDS and CODE nodes. You cannot read a node's intermediate output directly from the trace. To debug what a node produced, **echo it through a downstream LLM node**: add a temporary LLM with prompt `Output verbatim: {{$context.$nodes.<NODECODE>.$output}}` and read the END_USER response.
- **Tool responses:** `links[]` is stripped by Studio before the LLM sees it (see Portability gotcha above — relative paths are the fix). To verify a deployed agent's full toolset, grep the build artifact JSON — one trace shows only the tools Studio surfaced for that query (a subset). See the Portability / Reuse gotcha above.
- **Debug chat caches per (workflow, question)** — verify every redeploy with a never-asked question. See the Deploy gotcha above.
- **Runtime 401 after a pod refresh** is an auth/security issue (OPSS tables stale post-P2T), not a code issue. See the Runtime / Security gotcha above.

---

## Reference Implementation (optional)

If your project provides a known-good exemplar agent JSON (configured in the **Project Configuration** block above), diff new builds against it for import-shape correctness — particularly the `Specification` wrapper, tool nesting, and `agentMappings` edges.

If no exemplar exists, the JSON specifications in this skill are complete on their own. You do not need any external file to build a correct import JSON.

For example, a multi-agent reference build might be: a `data_pipeline` architecture with 1-token LLM router + SWITCH + REUSABLE module agents, all inner function names suffixed with the version number, domain-prefixed tool codes, `REST-Framework-Version: 9` headers on every BUS_OBJECT inner tool, per-resource IS-NOT-NULL sentinel filters, relative `/cs/idcplg` document links, no Source/Confidence footer, literal-example formatting in worker prompts, and block-card rendering with explicit `display:block` on every output element.
