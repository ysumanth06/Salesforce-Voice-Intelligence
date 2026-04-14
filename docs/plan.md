# Implementation Plan: Salesforce Voice Intelligence Layer

**Branch**: `feature/003-salesforce-voice-intelligence-layer` | **Date**: 2026-04-14 | **Spec**: [spec.md](./spec.md)
**Constitution**: [constitution.md](../../.sfspeckit/memory/constitution.md)

---

## Summary

The Voice Intelligence Layer adds voice recording, HTTP-chunked streaming transcription (via OpenAI Whisper), and AI-based task extraction to any Salesforce record page. The architecture extends `ContentVersion` with 5 custom fields, introduces two new LWC components (`voiceNoteRecorder`, `voiceIntelligencePanel`), and three Apex classes (`AudioRecorderController`, `StreamingTranscriptionController`, `VoiceTranscriptionService` Queueable). All OpenAI callouts are routed through a Named Credential; no direct browser API calls are made. This feature is isolated from the existing Smart Grid codebase — no shared classes or objects are modified.

---

## Technical Context

**Platform**: Salesforce Enterprise / Unlimited with Sales Cloud  
**API Version**: 65.0  
**CLI Version**: sf v2.x  
**Primary Languages**: Apex (server-side), JavaScript (LWC client-side)  
**Testing Framework**: Apex Test Framework (`sf apex run test`), Jest (LWC)  
**UI Framework**: Lightning Web Components with SLDS 2  
**Automation**: No Flow automation required (all orchestration is Apex-driven)  
**Deployment**: `sf project deploy start` with `--dry-run` validation  
**Source Format**: SFDX Source Format (decomposed metadata)  
**Integration**: OpenAI API (Whisper + GPT-4.1-mini / GPT-4.1) via Named Credential  
**Existing Permission Set**: `SmartGrid_User` — unaffected. New `Voice_Intelligence_User` PS created.

---

## Constitution Check

*GATE: Must pass before implementation begins.*

### Pre-Implementation Gates

#### Metadata-First Gate (Article I)
- [x] All objects/fields defined before any code references them — ContentVersion custom fields in Story-003-00
- [x] Permission Sets planned for all new fields — `Voice_Intelligence_User` PS in Story-003-00
- [x] Story-003-00 (Foundation) covers all shared metadata

#### Governor-Limit Gate (Article II)
- [x] Data volume estimates documented in spec — low-moderate volume, ~50 notes/user/day
- [x] Bulk scenarios identified — concurrent Queueables (≤50), flex queue saturation addressed
- [x] No SOQL/DML in loops — stateless per-chunk design; Queueable creates Tasks in single DML call

#### Declarative-First Gate (Article III)
- [x] Automation Approach Decision table completed in spec (9 behaviors evaluated)
- [x] Apex justified only where Flow/Validation Rules insufficient — all Apex use cases require async callouts or binary DML unavailable in Flow

#### Security Gate (Article IV)
- [x] `with sharing` default for all Apex classes
- [x] `WITH USER_MODE` for all SOQL (matching existing org pattern from `SmartGridController`)
- [x] No hardcoded Salesforce IDs in design
- [x] Named Credential for all OpenAI callouts — no API key in source

#### Test-First Gate (Article V)
- [x] PNB test pattern planned for all Apex
- [x] Test Data Factory extension planned (reuse existing patterns)
- [x] 90%+ coverage target set (NFR-004)

#### Separation of Concerns Gate (Article VI)
- [x] Service layer: `VoiceTranscriptionService` (business logic + LLM orchestration)
- [x] Controller layer: `AudioRecorderController`, `StreamingTranscriptionController` (`@AuraEnabled` only)
- [x] Selector pattern: SOQL encapsulated within service/controller, not in LWC or triggers
- [x] No business logic in LWC — components call Apex imperatively only

**Constitution Check Result: ✅ ALL GATES PASS**

---

## Salesforce Project Structure

```text
force-app/
└── main/
    └── default/
        ├── objects/
        │   └── ContentVersion/
        │       └── fields/
        │           ├── Transcription_Status__c.field-meta.xml     [NEW]
        │           ├── Transcript__c.field-meta.xml               [NEW]
        │           ├── Partial_Transcript__c.field-meta.xml       [NEW]
        │           ├── Streaming_Session_Id__c.field-meta.xml     [NEW]
        │           └── Error_Message__c.field-meta.xml            [NEW]
        │
        ├── classes/
        │   ├── AudioRecorderController.cls                        [NEW] @AuraEnabled — file save + Queueable enqueue
        │   ├── AudioRecorderController.cls-meta.xml               [NEW]
        │   ├── StreamingTranscriptionController.cls               [NEW] @AuraEnabled — stateless chunk relay
        │   ├── StreamingTranscriptionController.cls-meta.xml      [NEW]
        │   ├── VoiceTranscriptionService.cls                      [NEW] Queueable — Whisper + GPT pipeline
        │   ├── VoiceTranscriptionService.cls-meta.xml             [NEW]
        │   ├── VoiceTranscriptionServiceTest.cls                  [NEW] PNB tests — 90%+ coverage
        │   ├── VoiceTranscriptionServiceTest.cls-meta.xml         [NEW]
        │   ├── AudioRecorderControllerTest.cls                    [NEW]
        │   ├── AudioRecorderControllerTest.cls-meta.xml           [NEW]
        │   ├── StreamingTranscriptionControllerTest.cls           [NEW]
        │   └── StreamingTranscriptionControllerTest.cls-meta.xml  [NEW]
        │   (UNCHANGED: GridQueryBuilder, SmartGridController, SmartGridUserPrefService + tests)
        │
        ├── lwc/
        │   ├── voiceNoteRecorder/                                 [NEW]
        │   │   ├── voiceNoteRecorder.html
        │   │   ├── voiceNoteRecorder.js
        │   │   ├── voiceNoteRecorder.css
        │   │   ├── voiceNoteRecorder.js-meta.xml
        │   │   └── __tests__/
        │   │       └── voiceNoteRecorder.test.js
        │   └── voiceIntelligencePanel/                            [NEW]
        │       ├── voiceIntelligencePanel.html
        │       ├── voiceIntelligencePanel.js
        │       ├── voiceIntelligencePanel.css
        │       ├── voiceIntelligencePanel.js-meta.xml
        │       └── __tests__/
        │           └── voiceIntelligencePanel.test.js
        │   (UNCHANGED: smartDataGrid, smartGridFieldPicker)
        │
        ├── namedCredentials/
        │   └── OpenAI_API.namedCredential-meta.xml                [NEW]
        │
        ├── externalCredentials/
        │   └── OpenAI_External_Credential.externalCredential-meta.xml  [NEW]
        │
        └── permissionsets/
            └── Voice_Intelligence_User.permissionset-meta.xml     [NEW]
            (UNCHANGED: SmartGrid_User.permissionset-meta.xml)
```

---

## Impact Analysis

### Blast Radius Assessment

> SF CLI tooling API query was unavailable in this session. Impact assessed via source control analysis.

| Component Modified | Type | Existing Consumers | Risk Level | Mitigation |
|-------------------|------|-------------------|-----------|------------|
| `ContentVersion` (fields added) | Standard Object — field extension | DocGen package, standard file upload flows | 🟡 Medium | Use `[Voice]` title prefix; DocGen criteria-based filter. Validate DocGen trigger criteria in sandbox before deploying (GAP-006). |
| `SmartGridController` | Existing Apex Class | `smartDataGrid` LWC | 🟢 None | Not modified — zero blast radius. |
| `SmartGrid_User` Permission Set | Existing PS | Smart Grid users | 🟢 None | Not modified — zero blast radius. |
| `Task` (standard object) | Standard Object — no new fields | All Activity-related automations, reports | 🟢 Low | Tasks created via Apex with standard fields only. No schema change on Task. |
| New `Voice_Intelligence_User` PS | New PS | None yet | 🟢 None | Net new — no existing consumers. |

**Overall Blast Radius: 🟡 LOW-MEDIUM**  
Only risk is `ContentVersion` custom fields potentially triggering DocGen automation. Mitigated by `[Voice]` title prefix convention (FR-025). Requires one sandbox validation pass before production.

---

## Deployment Order

| Phase | Story | What to Deploy | SF Skill | Why This Order |
|-------|-------|---------------|----------|----------------|
| **1** | 003-00 | ContentVersion custom fields (5 fields) | sf-metadata | All Apex and LWC reference these — must exist first |
| **2** | 003-00 | Named Credential + External Credential (OpenAI) | sf-integration | Apex callouts require credentials to compile cleanly |
| **3** | 003-00 | `Voice_Intelligence_User` Permission Set | sf-metadata | FLS requires fields from Phase 1 to exist |
| **4** | 003-01 | `AudioRecorderController` + test | sf-apex | Controller saves ContentVersion; must exist before LWC |
| **5** | 003-02 | `StreamingTranscriptionController` + test | sf-apex | Streaming LWC wires to this controller |
| **6** | 003-03 | `VoiceTranscriptionService` (Queueable) + test | sf-apex | Called by AudioRecorderController; must be deployed together or after |
| **7** | 003-04 | `voiceNoteRecorder` LWC | sf-lwc | Depends on Phase 4 + 5 Apex controllers |
| **8** | 003-05 | `voiceIntelligencePanel` LWC | sf-lwc | Depends on Phase 4 Apex + Phase 7 LWC (sibling) |
| **9** | 003-06 | Integration test pass + DocGen validation | sf-testing | Full end-to-end verification, DocGen conflict check |

---

## Scoring Gates

| Artifact | SF Skill | Max Score | Min to Proceed | Block If Below |
|----------|----------|-----------|----------------|----------------|
| ContentVersion field XML | sf-metadata | 120 | 84 (70%) | 72 |
| Named Credential XML | sf-metadata | 120 | 84 (70%) | 72 |
| Permission Set XML | sf-metadata | 120 | 84 (70%) | 72 |
| `AudioRecorderController` | sf-apex | 150 | 90 (60%) | 75 |
| `StreamingTranscriptionController` | sf-apex | 150 | 90 (60%) | 75 |
| `VoiceTranscriptionService` | sf-apex | 150 | 90 (60%) | 75 |
| `voiceNoteRecorder` LWC | sf-lwc | 165 | 125 (76%) | 100 |
| `voiceIntelligencePanel` LWC | sf-lwc | 165 | 125 (76%) | 100 |
| All Apex Test Classes | sf-testing | 120 | 108 (90%) | 84 |

---

## Apex Architecture (Separation of Concerns)

### Layer Map

```
LWC (Client)
  │
  ├─► AudioRecorderController        [Controller Layer — @AuraEnabled]
  │     - saveAudioChunk(base64, recordId, title)  → inserts ContentVersion, enqueues VoiceTranscriptionService
  │     - validateFileSize(sizeBytes)              → enforces 10MB gate (FR-022)
  │
  ├─► StreamingTranscriptionController [Controller Layer — @AuraEnabled]
  │     - transcribeChunk(base64Chunk, sessionId)  → stateless callout to Whisper; returns partial transcript
  │
  └─► VoiceTranscriptionService (Queueable) [Service + Async Layer]
        - execute(QueueableContext ctx)
          1. Query ContentVersion by Id
          2. Whisper callout → full transcript
          3. GPT-4.1-mini task extraction → JSON
          4. Validate JSON schema (retry once → fallback GPT-4.1)
          5. Insert Tasks (max 10, WhatId = linkedEntityId)
          6. Update ContentVersion fields (Transcript__c, Status, Error)
```

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `StreamingTranscriptionController` is stateless | Each LWC chunk is a separate HTTP callout; Apex cannot maintain open connections. Session ID passed as parameter. |
| Single `VoiceTranscriptionService` Queueable (not Batch) | One recording = one file = one Queueable job. Batch is overkill; future bulk-replay can use Batch if needed. |
| `AudioRecorderController` validates file size | Server-side safety net (FR-022) — catches any bypass of LWC 5-min timer. |
| `with sharing` on all classes | Follows Constitution Article IV and org pattern (see `SmartGridController`). |
| `WITH USER_MODE` on all SOQL | Follows org pattern established in `SmartGridController`. |
| Max 10 tasks enforced in `VoiceTranscriptionService` | List truncation in Apex before DML (FR-023). |
| `ActivityDate` defaults to Today + 7 | Set in `VoiceTranscriptionService` when LLM returns null dueDate (FR-024). |

---

## LWC Architecture

### `voiceNoteRecorder`

| Property | Value |
|---------|-------|
| Target | `lightning__RecordPage` (any SObject) |
| API Props | `@api recordId` |
| Apex Calls | `AudioRecorderController.saveAudioChunk` (imperative), `StreamingTranscriptionController.transcribeChunk` (imperative, per chunk) |
| State | `isRecording`, `partialTranscript`, `timer` (counts to 300s), `mediaRecorder`, `audioChunks` |
| Key Behaviour | `MediaRecorder.start(1000)` → `ondataavailable` → base64 encode → `transcribeChunk` → append partial transcript. Auto-stop at 300s. |

### `voiceIntelligencePanel`

| Property | Value |
|---------|-------|
| Target | `lightning__RecordPage` (any SObject) |
| API Props | `@api recordId` |
| Wire Calls | None (imperative polling for real-time status; LDS refresh on record change) |
| Apex Calls | `AudioRecorderController.getVoiceNotes(recordId)` (imperative, on load + after recording) |
| Sections | Live Transcript (collapsible), Tasks Table, Pipeline Status Badge |
| Pagination | Most recent note expanded; older collapsed. "Load More" after 10. |
| Edit | Transcript edit gated by `currentUserId === ownerId` check in JS |

---

## Environment Strategy

| Environment | SF CLI Alias | Purpose | Deployed By |
|------------|-------------|---------|-------------|
| Dev Sandbox | `sandbox-org` | Development + code review | Developer |
| QA | (future) | Regression + DocGen validation | TPO |
| Production | (future) | Live release | TPO with `--dry-run` |

---

## Data Model

*See [data-model.md](./data-model.md) for full field definitions, XML examples, and ERD.*

### ContentVersion — New Custom Fields Summary

| Field API Name | Type | Required | Key Constraint |
|---------------|------|----------|----------------|
| `Transcription_Status__c` | Picklist | Yes | Values: Recording, Streaming, Finalizing, Extracting Tasks, Completed, Failed, Queued — Pending Capacity |
| `Transcript__c` | Long Text Area (131072) | No | Populated after final transcription |
| `Partial_Transcript__c` | Long Text Area (32768) | No | Updated per-chunk during streaming |
| `Streaming_Session_Id__c` | Text (255) | No | OpenAI session reference |
| `Error_Message__c` | Long Text Area (32768) | No | Populated on any failure state |

**ContentVersion OWD**: Follows ContentDocument sharing (inherited from parent record).

---

## Complexity Tracking

| Exception | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|--------------------------------------|
| Apex over Flow for Task creation | JSON parsing + conditional retry + DML in async context | Flow cannot parse dynamic JSON, cannot do conditional callout retry, cannot run in async Queueable |
| Queueable over Platform Event | Stateful job (needs ContentVersion Id + LinkedEntityId in context) | Platform Events are fire-and-forget; subscriber would need separate SOQL to recover context |
| Imperative polling in LWC over Wire | `Transcription_Status__c` changes are driven by an async Queueable; LDS wire adapters don't auto-refresh from server-side DML in background jobs | Wire adapters refresh on UI-layer navigation, not background Apex updates |

---

## Estimation Summary

| Layer | Complexity | Estimated Effort |
|-------|-----------|-----------------|
| Metadata (5 ContentVersion fields + Named Cred + PS) | Low | 2 hours |
| Apex — `AudioRecorderController` | Medium | 3 hours |
| Apex — `StreamingTranscriptionController` | Medium | 2 hours |
| Apex — `VoiceTranscriptionService` (Queueable + GPT pipeline + retry) | High | 6 hours |
| Apex — Test classes (PNB for all 3 classes) | High | 5 hours |
| LWC — `voiceNoteRecorder` (MediaRecorder + chunking + live transcript) | High | 6 hours |
| LWC — `voiceIntelligencePanel` (3 sections + pagination + edit guard) | Medium | 4 hours |
| LWC — Jest tests | Medium | 3 hours |
| Integration validation (DocGen + end-to-end recording test) | Low | 2 hours |
| **Total** | | **≈ 33 hours / 13 story points** |

---

## Research Notes

### OpenAI Whisper + GPT-4.1 via Named Credentials

Whisper accepts multipart/form-data uploads. In Apex, the audio file is retrieved from `ContentVersion.VersionData` (Blob) and sent as a base64-encoded body. The Named Credential endpoint is `https://api.openai.com`. Two separate callouts are made per Queueable job:
1. `POST /v1/audio/transcriptions` (Whisper model: `whisper-1`)
2. `POST /v1/chat/completions` (model: `gpt-4.1-mini` or `gpt-4.1`)

Both use `Bearer` token auth managed by External Credential principal — zero hardcoded keys.

### `StreamingTranscriptionController` — Chunk Strategy

Since Apex cannot hold open WebSocket connections, "streaming" in v1 is simulated: LWC calls `transcribeChunk(base64, sessionId)` for each 1-second audio chunk. The controller makes a synchronous Whisper callout per chunk and returns the partial transcript immediately. The LWC appends this to `partialTranscript` state. Full final transcription is done in the Queueable post-recording for accuracy.

### DocGen ContentVersion Risk

DocGen commonly uses process automation triggered on `ContentVersion.IsMajorVersion` or `ContentVersion.FileType` criteria. The `[Voice]` title prefix (FR-025) allows DocGen admins to add a title exclusion filter without code changes. Validate by running: `sf metadata list -m Flow --target-org sandbox-org` and inspecting any `ContentVersion`-triggered flows/processes.

---

## 🏛️ Architect Sign-Off

### Data Model Review
- [ ] Object relationships are correct (cardinality, cascade delete behavior)
- [ ] Field types are appropriate (no Text where Picklist is better)
- [ ] Naming conventions follow org standards
- **Reviewed by**: _______________
- **Notes**: _______________

### Security Review
- [ ] OWD settings are correct for each object
- [ ] Sharing rules are sufficient for cross-team access
- [ ] FLS approach uses Permission Sets (not Profiles)
- [ ] No over-permissioning (least privilege principle)
- **Reviewed by**: _______________

### Governor Limit Assessment
- [ ] Bulk scenarios identified and accounted for
- [ ] Query patterns will scale to estimated data volumes
- [ ] Async processing identified where needed (Batch, Queueable, Future)
- [ ] No known limit risks
- **Reviewed by**: _______________

### Integration Review
- [ ] Named Credentials configured for external callouts
- [ ] Retry and error handling patterns defined
- [ ] Callout timeout thresholds set
- [ ] Mock patterns defined for testing (HttpCalloutMock)
- **Reviewed by**: _______________

### Deployment Strategy
- [ ] Deployment order validated (9-phase plan above)
- [ ] No circular dependencies
- [ ] DocGen conflict mitigated via `[Voice]` title prefix
- [ ] Rollback plan: revert ContentVersion fields via `sf project deploy start --manifest` with previous package.xml
- **Reviewed by**: _______________

**Overall Sign-Off**: [ ] APPROVED / [ ] CHANGES REQUIRED  
**Date**: _______________

---

## Change Log

| Date | Author | Change |
|------|--------|--------|
| 2026-04-14 | TPO | Initial plan created from spec.md v1 (Clarified) |
