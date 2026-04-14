# Clarification Report: Salesforce Voice Intelligence Layer

**Feature**: `003-salesforce-voice-intelligence-layer`
**Report Date**: 2026-04-14
**Mode**: Interactive (Single-developer team — decisions recorded immediately)
**Spec**: `sfspeckit-data/specs/003-salesforce-voice-intelligence-layer/spec.md`
**Constitution**: `.sfspeckit/memory/constitution.md`
**Status**: ✅ All open items resolved — ready for `/sfspeckit-plan`

---

## Section 1: Org Consistency & Drift

### 1.1 Environment Baseline

| Item | Value |
|------|-------|
| Target Org Alias | `sandbox-org` |
| Org Config Source | `.sf/config.json` |
| SF CLI Available in Shell | ❌ Not found on PATH — CLI-based drift audit could not be executed automatically |
| Source-controlled Objects | `Smart_Grid_Column__mdt`, `Smart_Grid_Config__mdt`, `Smart_Grid_User_Pref__c` |

### 1.2 Drift Audit Results

> ⚠️ **SF CLI was not found on the shell PATH during automated drift analysis.** The CLI is installed (active org `sandbox-org` is configured in `.sf/config.json` and previous deployments are on record), but the binary was not accessible in this terminal session. A manual pre-build drift check is recommended before story implementation begins.

**Recommended manual commands to run before `/sfspeckit-plan`:**
```bash
sf metadata list -m CustomObject --target-org sandbox-org
sf project deploy preview --target-org sandbox-org
```

### 1.3 Known Source-Controlled Objects vs. Feature Requirements

| Object | In Source Control | Referenced by Feature 003 | Conflict Risk |
|--------|-------------------|--------------------------|---------------|
| `Smart_Grid_Column__mdt` | ✅ Yes | ❌ Not referenced | ✅ None |
| `Smart_Grid_Config__mdt` | ✅ Yes | ❌ Not referenced | ✅ None |
| `Smart_Grid_User_Pref__c` | ✅ Yes | ❌ Not referenced | ✅ None |
| `ContentVersion` | Standard (no local file) | ✅ Core object — 5 new custom fields | ⚠️ Confirm no conflicting Phase 1 ContentVersion fields exist in org |
| `Task` | Standard (no local file) | ✅ Used for task creation | ✅ None — no new fields on Task |

### 1.4 Drift Verdict

🟡 **Low Risk — Proceed with Caution**. No schema conflicts detected in source control. The only open concern is the `ContentVersion` custom fields: confirm Phase 1 (Feature 002) did not introduce conflicting fields on this object before Feature 003 stories begin. This is non-blocking for planning.

---

## Section 2: Technical Foundation — 10-Point Salesforce Checklist

| # | Check | Status | Notes |
|---|-------|--------|-------|
| 1 | **Object model defined** — all objects and relationships identified | ✅ Pass | ContentVersion, ContentDocumentLink, Task, source records. Custom fields listed. |
| 2 | **Security model defined** — OWD, FLS, sharing rules, Permission Sets | ✅ Pass | OWD accepted (ContentDocument sharing inherited). `Voice_Intelligence_User` Permission Set required. |
| 3 | **Governor limits addressed** — SOQL, DML, callouts, heap, CPU | ✅ Pass | Queueable callout budget (2–4), heap risk (chunked audio), CPU stateless design, flex queue limit all documented. |
| 4 | **Declarative-first evaluated** — Flow vs. Apex rationale documented | ✅ Pass | Automation Approach Decision table covers all 9 behaviors. Apex justified only where Flow cannot meet requirement. |
| 5 | **Error + fallback paths defined** — unhappy paths covered | ✅ Pass | Streaming failure → batch fallback. LLM failure → retry → GPT-4.1. Queue full → status field. |
| 6 | **External integration defined** — Named Credentials, network, auth | ✅ Pass | Named Credential + External Credential approach specified. No hardcoded keys. |
| 7 | **Test strategy defined** — PNB coverage, factory pattern | ✅ Pass | NFR-004 mandates ≥90% coverage. PNB pattern referenced from Constitution Article V. |
| 8 | **Deployment dependencies** — order of metadata deployment noted | ⚠️ Partial | Custom fields must deploy before Apex. Explicit deployment ordering deferred to `/sfspeckit-plan`. |
| 9 | **Mobile + browser compatibility** — UI verified for both | ✅ Pass | US-5 covers Salesforce Mobile. NFR-007 defines responsive breakpoints. MediaRecorder mobile caveat noted. |
| 10 | **Bulk scenarios addressed** — 200+ record batch considerations | ✅ Pass | Feature is user-initiated (1 recording = 1 ContentVersion). Bulk risk is concurrent Queueables, which is documented. |

**Checklist Score: 9.5 / 10** — Deployment ordering is the only partial item; resolved in plan phase.

---

## Section 3: Deep Business Analysis — Gap Findings

### 3.1 Previously Resolved Clarifications (from Spec)

All 6 original clarification items have been answered by the TPO and are locked in the spec. They are not re-opened here.

---

### 3.2 New Gaps Identified During Deep Analysis

---

#### GAP-001: Who Can Record? — No User Permission Boundary Defined

**Context**: The spec states the `Voice_Intelligence_User` Permission Set will control field-level access to `ContentVersion` custom fields. However, no permission boundary is specified for *who can initiate a recording* (i.e., the `voiceNoteRecorder` component itself). Any user with the LWC on their page layout could theoretically record.

**The Question**: Should all internal Salesforce users be able to record voice notes, or should recording be restricted to specific profiles or roles (e.g., Sales Reps and Service Agents only)?

**Proposed Options**:
- **Option A — Open to all internal users** (simple): No recording gate. Anyone who can see the LWC on their layout can record. Access controlled only by page layout assignment.
- **Option B — Permission Set gated** (recommended): The `Voice_Intelligence_User` Permission Set also controls *visibility* of the `voiceNoteRecorder` component, preventing unauthorized users from recording.

**Architectural Impact**: Option B requires a single additional metadata flag (component visibility via Permission Set). Adds minimal effort but provides a clean access control story.

**Decision**: ✅ **Option B** — Recording access controlled via `Voice_Intelligence_User` Permission Set.

---

#### GAP-002: What Happens to ContentVersion Files Over Time? — No Retention Policy

**Context**: The spec explicitly out-of-scopes file retention policy. However, with up to 50 voice notes per user per day, file storage will accumulate. A 5-minute audio file ≈ 9.5MB. With 50 users × 5 recordings/day × 250 working days = ~59GB/year.

**The Question**: Is there any storage concern or planned retention/deletion policy for voice note files?

**Proposed Options**:
- **Option A — No retention policy** (proceed as-is): Files accumulate indefinitely. Storage costs and limits managed by org admins separately.
- **Option B — Manual deletion only**: Users can delete their own ContentDocument records via standard Salesforce Files UI.
- **Option C — Auto-archive after N days** (complex): A scheduled Flow or Batch Apex deletes/archives ContentVersions older than N days.

**Architectural Impact**: Option A and B are zero additional build effort. Option C is a separate feature story.

**Decision**: ✅ **Option A** — No retention policy for this release. Flagged for future enhancement.

---

#### GAP-003: Transcript Editing — Who Can Edit?

**Context**: FR-015 states users SHOULD be able to edit the transcript and save back to `Transcript__c`. The spec does not define *who* can edit — only the recording user, or any user with access to the record?

**The Question**: If a manager or colleague can view the voice note on a shared record, can they also edit the transcript, or only the original recording user?

**Proposed Options**:
- **Option A — Any user with record access can edit** (simpler): No ownership check in the LWC edit action.
- **Option B — Only the recording user (ContentVersion owner) can edit** (recommended): LWC checks `CreatedById` of the ContentVersion against the current user before enabling the edit field.

**Architectural Impact**: Option B requires a single conditional check in the LWC (`currentUserId === contentVersion.CreatedById`). Low effort, high governance value.

**Decision**: ✅ **Option B** — Only the original recording user can edit the transcript.

---

#### GAP-004: AI Task Extraction — What Is "Actionable"?

**Context**: The spec states GPT-4.1-mini will extract "actionable tasks" from the transcript. The LLM prompt instructs it to return `[]` if no tasks exist. However, there is no guardrail defined for *over-extraction* — the LLM might interpret casual mentions ("I'll think about it") as tasks.

**The Question**: Should there be a maximum number of tasks that can be extracted from a single transcript, or is unlimited task creation acceptable?

**Proposed Options**:
- **Option A — No limit** (proceed as-is): Trust the LLM prompt to return only genuine tasks. No cap.
- **Option B — Cap at 10 tasks per transcript** (recommended): Apex enforces a maximum of 10 Task records per ContentVersion. Any additional items in the LLM response are silently dropped and logged in `Error_Message__c`.

**Architectural Impact**: A single Apex list size check — near-zero effort. Prevents edge case runaway Task creation from a verbose LLM response.

**Decision**: ✅ **Option B** — Maximum 10 Tasks per transcript enforced by Apex.

---

#### GAP-005: 5-Minute Limit Enforcement — UI or Backend?

**Context**: The spec now states the maximum recording duration is 5 minutes. The spec does not define *where* this limit is enforced — in the LWC (UI timer) or in the Apex controller (file size check), or both.

**The Question**: Should the 5-minute limit be enforced by the UI (auto-stopping the recording) or validated server-side as well?

**Proposed Options**:
- **Option A — UI only**: The `voiceNoteRecorder` LWC displays a timer and auto-stops `MediaRecorder` after 300 seconds. No server-side check.
- **Option B — UI + server-side** (recommended): LWC enforces the timer (user experience), and `AudioRecorderController` rejects ContentVersion inserts where the audio file exceeds 10MB (server-side safety net).

**Architectural Impact**: Option B adds a single size check in Apex. Protects against browser manipulation or future drift in the LWC.

**Decision**: ✅ **Option B** — 5-minute limit enforced in the LWC UI (timer) AND server-side by Apex (10MB file size gate).

---

#### GAP-006: DocGen Package Interaction — ContentVersion Trigger Risk

**Context**: The spec assumes DocGen does not interfere with ContentVersion. However, DocGen commonly uses `ContentVersion` triggers or process automations to generate documents, which could fire when voice note ContentVersions are inserted, causing unintended document generation side effects.

**The Question**: Does the DocGen package have any automation triggered on `ContentVersion` insert that could conflict with voice note files?

**Proposed Options**:
- **Option A — Assume no conflict** (accept risk): Proceed without investigation.
- **Option B — Add a record type or flag field** (recommended): Add a `Voice_Note__c` checkbox field to ContentVersion (or use `ContentVersion.Origin = 'H'` and a distinct `Title` prefix convention like `[Voice]`) so DocGen automations can be conditionally filtered to exclude voice note files.

**Architectural Impact**: Option B is a low-effort defensive measure — one additional field or a naming convention. Recommended to validate this with a manual check of DocGen trigger criteria in the sandbox before deploying.

**Decision**: ✅ **Option B** — Use a `[Voice]` title prefix convention for ContentVersions created by this feature. Validate DocGen trigger criteria in sandbox during the plan phase.

---

#### GAP-007: Task Due Date Handling — What If LLM Returns Null?

**Context**: The Task extraction prompt specifies `"dueDate": "YYYY-MM-DD or null"`. The spec does not define what Salesforce Task `ActivityDate` should be set to when the LLM returns `null` for `dueDate`.

**The Question**: When no due date is inferred from the transcript, what should the Task's due date be?

**Proposed Options**:
- **Option A — Leave `ActivityDate` blank**: Task is created with no due date.
- **Option B — Default to Today + 7 days** (recommended): A sensible business default — creates a follow-up deadline without requiring the user to manually update every undated task.
- **Option C — Leave blank, but surface in the panel**: Show undated tasks with a "No due date" badge in the `voiceIntelligencePanel` to prompt the user to set one.

**Architectural Impact**: All options are trivial to implement. Option B requires a single Apex date calculation. Option C requires a UI badge — slightly more LWC work.

**Decision**: ✅ **Option B** — Default `ActivityDate` to Today + 7 days when LLM returns `null` for `dueDate`.

---

#### GAP-008: Agentforce Readiness — Data Exposure Gap

**Context**: The constitution includes Article VIII (Agent Architecture). The spec does not address whether the Voice Intelligence data (transcripts, extracted tasks) should be exposed for Agentforce agents to consume or surface.

**The Question**: Is Agentforce integration (e.g., an agent topic that can query recent transcripts or trigger voice recording workflows) in scope for this feature or a future item?

**Proposed Options**:
- **Option A — Out of scope for this release**: Agentforce integration is a future feature.
- **Option B — Design for Agentforce readiness**: Ensure `ContentVersion` custom fields are marked as `externallyAccessible` where applicable, and document the object graph as a future Agentforce data source. No active Agentforce topics/actions built in this release.

**Architectural Impact**: Option B is zero additional build effort — it's a metadata flag and documentation note. Keeps the door open without adding scope.

**Decision**: ✅ **Option B** — Design for Agentforce readiness (metadata flags + documentation). Active Agentforce topics are out of scope for this release.

---

## Section 4: Stakeholder Sign-Off

| Role | Name | Status | Date |
|------|------|--------|------|
| TPO (Technical Product Owner) | Sumanth Yanamala | ✅ Approved (Interactive Mode) | 2026-04-14 |
| BPO (Business Process Owner) | — | N/A — Single-developer team | — |
| Architect | Sumanth Yanamala | ✅ Approved (Interactive Mode) | 2026-04-14 |

---

## Summary of All Decisions

| Gap | Decision |
|-----|---------|
| GAP-001: Who can record? | Recording gated by `Voice_Intelligence_User` Permission Set |
| GAP-002: File retention? | No retention policy for this release |
| GAP-003: Who can edit transcript? | Only the original recording user (ContentVersion owner) |
| GAP-004: Max tasks per transcript? | Maximum 10 Tasks per transcript, enforced by Apex |
| GAP-005: 5-min limit enforcement? | UI timer (LWC) + server-side 10MB file size gate (Apex) |
| GAP-006: DocGen conflict risk? | Use `[Voice]` title prefix convention; validate DocGen triggers in sandbox |
| GAP-007: Null due date handling? | Default `ActivityDate` to Today + 7 days when LLM returns null |
| GAP-008: Agentforce readiness? | Design for readiness (metadata flags); no active Agentforce topics this release |

---

## Next Step

✅ All gaps resolved. Proceed to:

```
/sfspeckit-plan
```

This will generate the technical implementation plan with story breakdown, deployment ordering, scoring gates, and Architect sign-off section.
