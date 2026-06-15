# NTT Compass — Oracle HCM Skills Export

Four reusable skills used to build and support the NTT Compass agents on Oracle Fusion HCM. Each folder is a self-contained skill (a `SKILL.md` plus any `references/`). Drop a folder into `~/.claude/skills/` (or a plugin's `skills/`) to install it.

| Skill | Purpose |
|---|---|
| **oracle-hcm-agent-builder** | Builds Oracle **AI Agent Studio** import JSON for Fusion HCM — agent teams, supervisor/worker patterns, document & business-object tools, workflow agents. The core "agentic" build skill behind the Compass agents. |
| **oracle-hcm-agentic-app-builder** | Builds import-ready **26B Agentic App / workspace** JSON — bundles agent teams into one swimlane workspace (Summary / Actions / Communications panels), wires member agents, authors the `oraInfoDisplay` / `oraComms` / `ora.Invoke` output blocks. Complements `oracle-hcm-agent-builder`. |
| **oracle-hcm-bip-extracts** | Builds/debugs **BI Publisher** reports (data models, `.xdm`/`.xdo`, catalog archives) and **HCM Extracts** (Extract Definition XML, user entities, fast-formula attributes, delivery options). Covers `_F`/`_TL`/`_VL`/`_B` table work and the BIP-vs-Extract decision. |
| **oracle-hcm-bip-template-builder** | Builds **BIP Excel templates** (`.xlsx`) for HCM extracts — Config Matrix / BR100 layouts, `XDO_METADATA`/`XDO_GROUP` bindings, repeating-row layout, cell-to-xpath tag binding. Template layout only (pairs with `oracle-hcm-bip-extracts`). |

**Install:** copy a skill folder into the CLI's `skills/` directory (e.g. `~/.claude/skills/`), then restart the CLI — skills are auto-discovered.

*Owned by NTT Data.*
