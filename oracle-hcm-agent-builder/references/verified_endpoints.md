# Oracle HCM REST API — Verified Endpoints

Verified April 2026 against https://docs.oracle.com/en/cloud/saas/human-resources/farws/rest-endpoints.html

## Full CRUD Resources (14 confirmed)

| Resource | Path | API |
|----------|------|-----|
| positions | /hcmRestApi/resources/11.13.18.05/positions | hcmRestApi |
| jobs | /hcmRestApi/resources/11.13.18.05/jobs | hcmRestApi |
| grades | /hcmRestApi/resources/11.13.18.05/grades | hcmRestApi |
| locations | /hcmRestApi/resources/11.13.18.05/locations | hcmRestApi |
| organizations | /hcmRestApi/resources/11.13.18.05/organizations | hcmRestApi |
| gradeLadders | /hcmRestApi/resources/11.13.18.05/gradeLadders | hcmRestApi |
| gradeRates | /hcmRestApi/resources/11.13.18.05/gradeRates | hcmRestApi |
| jobFamilies | /hcmRestApi/resources/11.13.18.05/jobFamilies | hcmRestApi |
| goalPlans | /hcmRestApi/resources/11.13.18.05/goalPlans | hcmRestApi |
| recruitingCEJobRequisitions | /hcmRestApi/resources/11.13.18.05/recruitingCEJobRequisitions | hcmRestApi |
| valueSets | /fscmRestApi/resources/11.13.18.05/valueSets | fscmRestApi |
| commonLookups | /fscmRestApi/resources/11.13.18.05/commonLookups | fscmRestApi |
| setupTasks | /fscmRestApi/resources/11.13.18.05/setupTasks | fscmRestApi |
| DFF Contexts | /fscmRestApi/.../FlexFndDffDescriptiveFlexfieldContexts | fscmRestApi |

## LOV-Only Endpoints (read-only lookups, 14+ confirmed)

These exist but are List of Values endpoints — read-only, no CRUD:

| Resource | Path |
|----------|------|
| departmentsLov | /hcmRestApi/resources/11.13.18.05/departmentsLov |
| departmentsLovV2 | /hcmRestApi/resources/11.13.18.05/departmentsLovV2 |
| personTypesLOV | /hcmRestApi/resources/11.13.18.05/personTypesLOV |
| actionsLOV | /hcmRestApi/resources/11.13.18.05/actionsLOV |
| actionReasonsLOV | /hcmRestApi/resources/11.13.18.05/actionReasonsLOV |
| payrollDefinitionsLOV | /hcmRestApi/resources/11.13.18.05/payrollDefinitionsLOV |
| payrollBalanceDefinitionsLOV | /hcmRestApi/resources/11.13.18.05/payrollBalanceDefinitionsLOV |
| salaryBasisLov | /hcmRestApi/resources/11.13.18.05/salaryBasisLov |
| eligibilityProfilesLOV | /hcmRestApi/resources/11.13.18.05/eligibilityProfilesLOV |
| benefitPlansLOV | /hcmRestApi/resources/11.13.18.05/benefitPlansLOV |
| absenceTypesLOV | /hcmRestApi/resources/11.13.18.05/absenceTypesLOV |
| successionPlansLOV | /hcmRestApi/resources/11.13.18.05/successionPlansLOV |
| hcmBusinessUnitsLOV | /hcmRestApi/resources/11.13.18.05/hcmBusinessUnitsLOV |
| legalEmployersLov | /hcmRestApi/resources/11.13.18.05/legalEmployersLov |
| hcmTreesLOV | /hcmRestApi/resources/11.13.18.05/hcmTreesLOV |
| assignmentStatusTypesLov | /hcmRestApi/resources/11.13.18.05/assignmentStatusTypesLov |
| performanceTemplateDocumentNamesLOV | /hcmRestApi/resources/11.13.18.05/performanceTemplateDocumentNamesLOV |

## NOT FOUND — No REST Endpoint Exists (16)

These config objects do NOT have REST APIs. Use Document Tool RAG with FSM CSV exports instead:

departments (standalone CRUD), elements, elementLinks, payrolls (config detail), fastFormulas, compensationPlans, accrualPlans, performanceTemplates, candidateSelectionProcesses, learningItems, learningRecordSpecializations, hrHelpDeskCategories, hrHelpDeskKnowledgeArticles, extensibleFlexfields, approvalRules, workforceStructures

## Common Mistakes to Avoid

- `gradeRateValues` → use `gradeRates` (correct resource name)
- `setupTaskListDetails` → use `setupTasks` (correct resource name)
- `lookups` → use `commonLookups` (different resource name, under fscmRestApi)
- `departments` → use `departmentsLov` (no standalone CRUD endpoint for departments)
- Never use `/latest/` in paths — always use versioned: `/11.13.18.05/`
