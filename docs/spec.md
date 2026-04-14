# Feature Specification: Salesforce Voice Intelligence Layer

**Feature Branch**: `feature/003-salesforce-voice-intelligence-layer`
**Created**: 2026-04-14
**Status**: Clarified ✅
**Author**: TPO
**Input**: User description: "Build a production-ready Salesforce Voice Intelligence Layer — voice recording on any record, real-time streaming transcription, AI-based task extraction, inline UI dashboard showing transcript + tasks, and automatic Task creation."

---

## Salesforce Platform Context

**Target Org Type**: Sandbox  
**Target Clouds**: Sales Cloud (with compatibility across Service Cloud, custom objects)  
**Target Editions**: Enterprise / Unlimited  
**API Version**: 65.0  
**sf CLI Version**: v2.x  
**Existing Packages**: DocGen  
**Org Constraints**: Must comply with callout limits (100/transaction), Apex heap limits; WebSocket not supported natively in Apex — browser-side only via LWC. Named Credentials required for OpenAI API access.

---

## User Stories

### User Story 1 — Voice Recording on Any Record (Priority: P1)

As any Salesforce user (Sales Rep, Service Agent, Account Manager), I want to record a voice note directly on any standard or custom object record page so that I can capture conversation context without leaving Salesforce.

**Why this priority**: This is the foundational capability. Without voice capture, no downstream feature (transcription, task extraction) can function. It is the entry point for the entire Voice Intelligence Layer.

**Independent Test**: Can be verified by navigating to any Account record, placing the `voiceNoteRecorder` LWC on the page layout, clicking "Start Recording", capturing audio, and confirming the file is saved as a ContentVersion linked to the record.

**Acceptance Scenarios**:

1. **Given** a user is on any standard (Account, Contact, Opportunity, Case) or custom object record page, **When** they click "Start Recording" on the `voiceNoteRecorder` component, **Then** the browser requests microphone permission and begins capturing audio via the MediaRecorder API.
2. **Given** a recording is in progress, **When** the user clicks "Stop", **Then** the audio file is uploaded to Salesforce as a ContentVersion linked to the record via ContentDocumentLink.
3. **Given** the user has no microphone access, **When** they click "Start Recording", **Then** a clear error message is displayed and no recording session is initiated.
4. **Given** a recording session is active, **When** the user navigates away, **Then** the recording is gracefully stopped and the audio is saved before navigation completes.

---

### User Story 2 — Real-Time Streaming Transcription (Priority: P1)

As a Salesforce user actively recording a voice note, I want to see the transcript appear in near real-time on the screen so that I can verify capture accuracy and correct any errors immediately during or after the session.

**Why this priority**: Real-time feedback is the primary differentiator of this feature over batch transcription. It directly impacts user trust and adoption.

**Independent Test**: Can be verified by starting a recording session, speaking clearly, and confirming that partial transcript text appears in the live transcript panel within 2–3 seconds of speech, streaming progressively as audio chunks are sent.

**Acceptance Scenarios**:

1. **Given** a recording is in progress using the MediaRecorder `timeslice(1000)` API, **When** each 1-second audio chunk is produced, **Then** the LWC sends the base64-encoded chunk to `StreamingTranscriptionController` and the returned partial transcript is appended to the live transcript display area.
2. **Given** the streaming API (OpenAI Realtime) is available, **When** audio chunks are streamed, **Then** partial `transcript.delta` JSON responses are received and progressively displayed.
3. **Given** the OpenAI Realtime API is unavailable or times out, **When** audio chunks are buffered, **Then** the system falls back to Whisper batch transcription using the `TranscriptionService` Queueable after recording ends.
4. **Given** the recording has ended, **When** the final transcription is complete, **Then** `Transcript__c` on the ContentVersion is set to the final transcript and `Transcription_Status__c` is set to "Completed".
5. **Given** a streaming failure occurs mid-session, **When** the fallback is triggered, **Then** `Transcription_Status__c` is set to "Processing" and a "Finalizing..." indicator is shown in the UI.

---

### User Story 3 — AI-Based Task Extraction (Priority: P1)

As a Salesforce user, after a voice note is transcribed, I want the system to automatically extract actionable tasks from the transcript and create them as Salesforce Task records linked to the current record so that I don't need to manually create follow-up actions.

**Why this priority**: Task extraction delivers direct business value — converting unstructured conversation into structured CRM actions. This is the core ROI driver of the feature.

**Independent Test**: Can be verified by providing a transcript containing phrases like "call John tomorrow" or "send the proposal by Friday" and confirming that corresponding Task records are auto-created with appropriate Subject, Description, Priority, and Due Date, linked via `WhatId`.

**Acceptance Scenarios**:

1. **Given** a final transcript is available on a ContentVersion, **When** `TranscriptionService` invokes the GPT-4.1-mini task extraction prompt, **Then** the LLM returns a valid JSON array of task objects matching the defined schema.
2. **Given** the extracted JSON is valid, **When** the Apex service processes the response, **Then** one Salesforce Task record is created per extracted task item, with `WhatId` set to the source record's Id, and `ActivityDate`, `Priority`, `Subject`, and `Description` populated.
3. **Given** the LLM returns invalid JSON on the first attempt, **When** Apex retries, **Then** a second call is made with the same prompt; if still invalid, the call falls back to GPT-4.1.
4. **Given** the transcript contains no actionable tasks, **When** the LLM is invoked, **Then** it returns `[]` and no Task records are created; `Transcription_Status__c` is still set to "Completed".
5. **Given** all LLM calls fail, **When** the error state is reached, **Then** `Transcription_Status__c` is set to "Failed", `Error_Message__c` is populated, and no Tasks are silently lost.

---

### User Story 4 — Inline Voice Intelligence Dashboard (Priority: P2)

As a Salesforce user, I want a single embedded dashboard on any record page that shows the live/final transcript, extracted tasks, and the current processing pipeline status so that I have full visibility into the voice intelligence workflow without navigating away.

**Why this priority**: The dashboard ties all three core capabilities together into a unified UX. Without it, users would need to navigate to Tasks or ContentVersions separately.

**Independent Test**: Can be verified by placing `voiceIntelligencePanel` on an Account page, completing a full voice recording, and confirming that the panel displays the real-time transcript, the extracted Task table, and the correct pipeline status badge without a page refresh.

**Acceptance Scenarios**:

1. **Given** `voiceIntelligencePanel` is placed on a record page, **When** the page loads, **Then** the panel queries ContentVersions linked to the record and displays the most recent transcript and its status.
2. **Given** a recording is in progress, **When** partial transcripts arrive, **Then** the live transcript area auto-scrolls to the bottom and highlights newly appended text.
3. **Given** the transcript is editable, **When** a user modifies the text area, **Then** changes are saved back to `Transcript__c` on the ContentVersion on blur or explicit save action.
4. **Given** tasks have been extracted, **When** the Tasks section renders, **Then** a tabular list shows Subject, Description, Priority, Due Date, and Status; clicking any row opens the Task record.
5. **Given** the pipeline is processing, **When** `Transcription_Status__c` changes, **Then** the Process Status section reflects the current stage (Recording → Streaming → Finalizing → Extracting → Completed/Failed) without a full page reload.

---

### User Story 5 — Cross-Object Compatibility & Mobile Support (Priority: P3)

As a Salesforce Administrator, I want the Voice Intelligence Layer components to be configurable on any standard or custom object record page and function correctly on Salesforce Mobile so that the feature can be deployed org-wide without per-object customization.

**Why this priority**: Cross-object and mobile compatibility broadens the feature's addressable use base but is not required for initial value delivery on a single object.

**Independent Test**: Can be verified by adding `voiceNoteRecorder` and `voiceIntelligencePanel` to a Case record page layout, completing a recording on mobile, and confirming the transcript and tasks appear correctly.

**Acceptance Scenarios**:

1. **Given** the LWC components are added to a custom object page layout, **When** a user navigates to that record, **Then** the components render and function identically to their behavior on standard objects.
2. **Given** the Salesforce Mobile App is used, **When** the user initiates a recording, **Then** the MediaRecorder API functions within the mobile browser context and audio is captured without UI freeze.
3. **Given** a mobile recording is completed, **When** the user returns to the record, **Then** the transcript and tasks are visible in `voiceIntelligencePanel` with no layout overflow or broken styles.

---

## Automation Approach Decision

| Behavior | Approach | Justification |
|----------|----------|---------------|
| Audio capture and chunk streaming | LWC (`voiceNoteRecorder`) | Browser MediaRecorder API requires client-side JS; no Salesforce declarative equivalent |
| Sending audio chunks to transcription | Apex Controller (`StreamingTranscriptionController`) | External callout required; LWC cannot make direct API calls securely |
| Saving audio file to Salesforce | Apex Controller (`AudioRecorderController`) | ContentVersion insert with binary data requires Apex DML; not available in Flow |
| Final transcription fallback | Queueable Apex (`TranscriptionService`) | Async processing required to avoid Apex transaction callout limits; Queueable is the appropriate async pattern |
| GPT task extraction and retry logic | Apex Service (inside `TranscriptionService`) | JSON schema validation, conditional retry, and multi-model fallback require imperative logic — Flow cannot handle |
| Salesforce Task record creation | Apex (inside `TranscriptionService`) | DML within async context; Flow could handle simple creation but combined JSON parsing + DML requires Apex |
| Viewing extracted tasks on record page | LWC (`voiceIntelligencePanel`) | Dynamic, auto-refreshing UI with real-time status — Screen Flow lacks the required reactive wire adapter patterns |
| Pipeline status updates | ContentVersion field (`Transcription_Status__c`) + LDS wire adapter | Lightweight status propagation leverages Lightning Data Service reactivity; no Flow needed |
| Named Credential provisioning | Named Credential + External Credential metadata | Declarative credential management; no custom Apex needed for authentication config |

---

## Salesforce Object Map

| Object | Type | Relationship | Notes |
|--------|------|-------------|-------|
| ContentVersion | Standard | — | Stores audio file; extended with custom fields for transcript, partial transcript, status, session ID, error |
| ContentDocumentLink | Standard | Lookup → ContentDocument, Lookup → Source Record | Links content to any SObject record via `LinkedEntityId` |
| Task | Standard | Lookup → Source Record via `WhatId` | Auto-created from extracted tasks; linked to source record |
| ContentDocument | Standard | Parent of ContentVersion | Created automatically when ContentVersion is inserted |
| Account / Contact / Opportunity / Case / Custom Object | Standard / Custom | Source records only; no new fields on these objects | Components embedded via page layout; record ID passed as `@api recordId` |

**New Custom Fields on ContentVersion**:
| Field API Name | Type | Purpose |
|---------------|------|---------|
| `Transcription_Status__c` | Picklist (Recording, Streaming, Finalizing, Extracting Tasks, Completed, Failed) | Pipeline status tracking |
| `Transcript__c` | Long Text Area (131072) | Final transcript stored here |
| `Partial_Transcript__c` | Long Text Area (32768) | Running partial transcript during streaming |
| `Streaming_Session_Id__c` | Text (255) | OpenAI Realtime session ID for stateful streaming |
| `Error_Message__c` | Long Text Area (32768) | Stores error details on failure |

---

## Data Volume & Governor Limit Considerations

- **Expected record volume**: ContentVersion records — low to moderate volume per org (estimate 5–50 voice notes per user per day); Task records — moderate (1–5 per voice note)
- **Bulk operation scenarios**: Not a bulk-record-creation feature; each recording is a single user action. No batch DML patterns needed for this feature's primary path.
- **Peak concurrent users**: Up to 50 users recording simultaneously; each triggers independent Queueable jobs — must stay within Queueable flex queue limits (100 queued jobs)
- **Limits to monitor**:
  - **Callouts**: Each Queueable job may make 2–4 callouts (Whisper + GPT retry + GPT fallback). Must stay ≤ 100 callouts/transaction.
  - **Heap size**: Audio chunks processed as base64 strings. Large audio files (>4.5MB base64) may approach the 6MB heap limit. Implement chunked streaming to mitigate.
  - **CPU time**: Streaming controller must be stateless per chunk (no heavy parsing). Keep chunk processing ≤ 1000ms CPU.
  - **Concurrent Queueables**: Monitor flex queue saturation in high-concurrency scenarios.
- **Large data volume (LDV) concerns**: Not applicable for initial release. ContentVersion is not expected to exceed 1M records.

---

## Security & Sharing Model

- **Record-level access (OWD)**:
  - ContentVersion: Controlled by ContentDocument sharing (follows parent record sharing)
  - Task: Private (OWD); Tasks inherit access from related record owner and standard Activity sharing
- **Field-level security**: All five new ContentVersion fields must be accessible via a dedicated Permission Set (`Voice_Intelligence_User`) granted to intended users/profiles
- **Recording access**: The `voiceNoteRecorder` component is gated by the `Voice_Intelligence_User` Permission Set — only users with this Permission Set can record (GAP-001 decision).
- **Transcript editing**: Only the original recording user (ContentVersion owner) can edit the transcript in `voiceIntelligencePanel`; all other users with record access are read-only (GAP-003 decision).
- **Sharing rules**: No new custom sharing rules required; existing record sharing governs who can see linked voice notes and tasks
- **Record ownership**: ContentVersion owner = the recording user. Task owner = the recording user (matches Salesforce Activity default)
- **Named Credential**: `OpenAI_Named_Credential` must be configured with the OpenAI API key stored as an External Credential principal. No API key exposure in LWC or Apex source.
- **API key security**: LWC must never directly call OpenAI. All callouts must route through Apex using Named Credentials. Transcript text must be sanitized (no script injection) before storage.
- **Guest user access**: Not applicable — feature is for authenticated internal Salesforce users only.

---

## Edge Cases

### General Edge Cases
- What happens if the user's browser does not support `MediaRecorder`? → Display an unsupported browser warning; disable recording button; provide a fallback "record on desktop" message.
- What happens if the microphone permission is denied mid-session? → Catch the `NotAllowedError` in LWC; stop the timer; surface a user-friendly error; save any buffered audio before the denial.
- What if the OpenAI API is down when the Queueable runs? → Set `Transcription_Status__c` to "Failed" and `Error_Message__c` with the HTTP response body; do not re-queue (to avoid runaway retries).
- What if a user records multiple voice notes on the same record? → Each ContentVersion creates an independent pipeline; `voiceIntelligencePanel` shows the most recent by `CreatedDate` with the ability to browse all via timeline list.
- What if the transcript contains PII? → No masking required; standard Salesforce platform encryption is sufficient (GAP-002 resolved).
- What if a Task with the same subject already exists? → Always create a new Task record; no deduplication logic applied (Clarification #2 resolved).
- What if the LLM extracts more than 10 tasks from a single transcript? → Apex enforces a maximum of 10 Task records per ContentVersion; excess items are dropped and logged in `Error_Message__c` (GAP-004 decision).
- What if a user records for longer than 5 minutes? → The LWC auto-stops the recording after 300 seconds (5 min). If the file size exceeds 10MB, `AudioRecorderController` rejects the upload and surfaces a user-friendly error (GAP-005 decision).
- What if DocGen automations trigger on voice note ContentVersion inserts? → Voice note ContentVersions use the title prefix `[Voice]` to allow DocGen criteria-based filtering (GAP-006 decision). Validate DocGen trigger criteria in sandbox during plan phase.
- What if the LLM returns `null` for a task's due date? → `ActivityDate` defaults to Today + 7 days (GAP-007 decision).

### Governor Limit Edge Cases
- What happens if a single audio chunk base64-encodes to >1MB? → The `StreamingTranscriptionController` must enforce a chunk size maximum and return an error to the LWC if exceeded; the LWC should reduce `timeslice` or compress before sending.
- What happens if the Queueable flex queue is full (100 jobs)? → `System.enqueueJob()` will throw; `AudioRecorderController` must catch this exception and set `Transcription_Status__c` to "Queued — Pending Capacity".
- What if the GPT callout chain (primary + retry + fallback) consumes all 3 callouts in the Queueable? → Design `TranscriptionService` to track callout count; if near limit, abort gracefully and surface error.
- What if ContentVersion insert fails due to storage limits? → Catching the DML exception and surfacing a user-friendly "Storage full" message is required.

---

## Functional Requirements

- **FR-001**: System MUST allow users to start and stop voice recording from any standard or custom object record page via the `voiceNoteRecorder` LWC.
- **FR-002**: System MUST capture audio using the browser MediaRecorder API with `timeslice(1000)` to produce 1-second chunks.
- **FR-003**: System MUST send each audio chunk (base64-encoded) to `StreamingTranscriptionController` and return a partial transcript update to the LWC within 3 seconds per chunk.
- **FR-004**: System MUST attempt real-time streaming transcription via OpenAI Realtime API (WebSocket, browser-side); if unavailable, MUST fall back to Whisper batch transcription via Queueable Apex.
- **FR-005**: System MUST display the live partial transcript in the `voiceNoteRecorder` LWC transcript panel, automatically appending new text and scrolling to the latest entry.
- **FR-006**: System MUST save the completed audio recording as a ContentVersion linked to the source record via ContentDocumentLink.
- **FR-007**: System MUST invoke GPT-4.1-mini task extraction on the final transcript using the strict JSON schema prompt.
- **FR-008**: System MUST validate the extracted task JSON schema in Apex before creating Task records.
- **FR-009**: System MUST retry GPT-4.1-mini once on schema validation failure, then fall back to GPT-4.1.
- **FR-010**: System MUST create one Salesforce Task record per extracted task item, with `WhatId` set to the source record Id.
- **FR-011**: System MUST expose `voiceIntelligencePanel` as a standalone LWC embeddable on any record page, displaying: live/final transcript, extracted Tasks table, and pipeline status.
- **FR-012**: System MUST update `Transcription_Status__c` at each pipeline stage: Recording → Streaming → Finalizing → Extracting Tasks → Completed / Failed.
- **FR-013**: System MUST surface errors with clear user-visible messages; all failure states MUST populate `Error_Message__c`.
- **FR-014**: System MUST use Named Credentials for all OpenAI API calls; API keys MUST NOT be hardcoded in Apex or LWC.
- **FR-015**: System SHOULD allow users to manually edit the transcript in `voiceIntelligencePanel`; only the original recording user (ContentVersion owner) may edit and save corrections back to `Transcript__c` (GAP-003).
- **FR-016**: System SHOULD debounce streaming callouts to avoid hitting callout limits during rapid chunk production.
- **FR-017**: System MAY support OpenAI Realtime API WebSocket streaming natively if/when Salesforce enables browser-to-external WebSocket passthrough in LWC (future enhancement — not P1).
- **FR-018**: System MUST sanitize transcript content before storing to prevent XSS or injection attacks.
- **FR-019**: System MUST be compatible with Salesforce Mobile App on both iOS and Android browsers.
- **FR-020**: `Transcription_Status__c` field MUST use a picklist: `Recording`, `Streaming`, `Finalizing`, `Extracting Tasks`, `Completed`, `Failed`, `Queued — Pending Capacity` (Clarification #3 resolved).
- **FR-021**: Recording access MUST be gated by the `Voice_Intelligence_User` Permission Set; users without this Permission Set MUST NOT see the `voiceNoteRecorder` component (GAP-001).
- **FR-022**: System MUST enforce a 5-minute maximum recording duration: the LWC MUST auto-stop after 300 seconds, and `AudioRecorderController` MUST reject files exceeding 10MB with a user-friendly error message (GAP-005).
- **FR-023**: System MUST cap Task creation at a maximum of 10 Tasks per ContentVersion; any additional extracted items MUST be dropped and logged in `Error_Message__c` (GAP-004).
- **FR-024**: When the LLM returns `null` for a task's due date, `ActivityDate` MUST default to Today + 7 days (GAP-007).
- **FR-025**: All ContentVersions created by this feature MUST use the title prefix `[Voice]` to distinguish them from other Salesforce files (DocGen conflict mitigation — GAP-006).
- **FR-026**: `voiceIntelligencePanel` MUST display all historical voice notes for a record, with the most recent expanded by default, older entries collapsed, and a "Load More" control appearing after 10 entries (Clarification #4 resolved).
- **FR-027**: System MUST be architected such that `ContentVersion` custom fields are accessible to future Agentforce agents (Agentforce readiness — GAP-008); no active Agentforce topics are built in this release.

---

## Non-Functional Requirements

- **NFR-001**: Partial transcript updates MUST appear in the LWC within 3 seconds of each audio chunk being sent.
- **NFR-002**: `StreamingTranscriptionController` MUST be stateless per chunk invocation (no shared state between Apex calls).
- **NFR-003**: All Apex transactions MUST complete within 10,000ms CPU time; streaming controller invocations MUST complete within 3,000ms CPU.
- **NFR-004**: Code coverage MUST exceed 90% across all Apex classes (per Constitution Article V and scoring gate sf-apex: 90/150).
- **NFR-005**: LWC components MUST NOT freeze the browser UI during streaming; audio chunk processing MUST be handled asynchronously via `ondataavailable` events.
- **NFR-006**: System MUST handle up to 50 concurrent recording sessions without Queueable flex queue exhaustion.
- **NFR-007**: The `voiceNoteRecorder` and `voiceIntelligencePanel` components MUST be responsive and usable on screens from 320px (mobile) to 1920px (desktop).
- **NFR-008**: Apex heap usage per `StreamingTranscriptionController` invocation MUST remain below 4MB to ensure headroom for Salesforce internal overhead.

---

## Success Criteria

- **SC-001**: Users can complete a full voice-to-task cycle (record → transcription → task creation) in under 60 seconds for a 30-second audio input.
- **SC-002**: Partial transcript text appears in the LWC within 3 seconds of each 1-second audio chunk being sent.
- **SC-003**: 95%+ of GPT task extraction calls produce valid JSON on the first or second attempt.
- **SC-004**: The Voice Intelligence Layer works correctly on at least 3 distinct object types (Account, Opportunity, and one custom object) without configuration changes.
- **SC-005**: All Apex test classes achieve ≥ 90% code coverage with PNB (Positive/Negative/Bulk) test patterns per Constitution Article V.
- **SC-006**: Zero API keys exposed in source code or LWC bundled assets at any time.
- **SC-007**: Users report the live transcript view accurately reflects spoken content with less than 5% word error rate on clear speech.

---

## Assumptions & Dependencies

- **A1**: The target Salesforce org has Connected App / Named Credential infrastructure available to configure OpenAI API access (HTTP Named Credential for Whisper/GPT, External Credential for API key storage).
- **A2**: The OpenAI Realtime API WebSocket approach will be implemented browser-side only within the LWC (Salesforce Apex cannot maintain WebSocket connections); Apex serves as the HTTP bridge for chunk-by-chunk fallback only.
- **A3**: Audio recordings will be stored as ContentVersion/ContentDocument files, which counts against Salesforce file storage limits. File retention policy is not in scope for this feature.
- **A4**: The feature targets authenticated internal Salesforce users only; no Experience Cloud / Guest User access is required for this release.
- **A5**: The team has access to an active OpenAI API key with access to `gpt-4.1-mini`, `gpt-4.1`, and Whisper models.
- **A6**: The existing Smart Grid objects (`Smart_Grid_Column__mdt`, `Smart_Grid_Config__mdt`, `Smart_Grid_User_Pref__c`) are unaffected by this feature; no schema conflicts are anticipated.
- **A7**: DocGen (installed package) does not interfere with ContentVersion custom fields or ContentDocumentLink behavior.
- **D1**: Dependency on `sf-metadata` skill to create ContentVersion custom fields and the `Voice_Intelligence_User` Permission Set.
- **D2**: Dependency on `sf-lwc` skill to scaffold `voiceNoteRecorder` and `voiceIntelligencePanel` components.
- **D3**: Dependency on `sf-apex` skill to build `AudioRecorderController`, `StreamingTranscriptionController`, and `TranscriptionService`.
- **D4**: Dependency on `sf-integration` skill to configure Named Credential and External Credential for OpenAI.
- **D5**: Dependency on `sf-testing` skill to generate and validate PNB test classes.

---

## Out of Scope

- Real-time speaker diarization (identifying multiple speakers in the transcript).
- Audio playback of stored recordings within Salesforce.
- Automatic language detection or multi-language transcription support.
- Custom AI model fine-tuning or hosting (feature relies entirely on OpenAI hosted models).
- Transcript search, indexing, or cross-record analytics dashboard.
- Salesforce Einstein Voice (deprecated) integration.
- PII masking or data residency enforcement — not required (Clarification #1 resolved).
- Task deduplication logic — always create new Tasks (Clarification #2 resolved).
- Automatic file retention / deletion policy — deferred to future release (GAP-002).
- Active Agentforce topics or actions for voice data — deferred to future release (GAP-008).
- Integration with external calendar or task management systems (e.g., Google Tasks, Jira).
- OpenAI Realtime API WebSocket streaming — deferred to future release (Clarification #5 resolved).

---

## Clarification Status

| # | Question | Status | Answer |
|---|----------|--------|--------|
| 1 | Should transcripts containing PII be masked, encrypted beyond platform encryption, or subject to data residency constraints? | ✅ Answered | No — standard Salesforce platform encryption is sufficient; no PII masking required. |
| 2 | Should the Task creation process deduplicate against existing Tasks on the same record (same subject/due date), or always create new records? | ✅ Answered | Always create new Task records — no deduplication logic required. |
| 3 | Should `Queued — Pending Capacity` be added as a formal `Transcription_Status__c` picklist value for Queueable queue-full scenarios? | ✅ Answered | Yes — add `Queued — Pending Capacity` as a valid picklist value. |
| 4 | Should the `voiceIntelligencePanel` display all historical voice notes for a record (with pagination) or only the most recent one? | ✅ Answered | Show all voice notes. Most recent expanded by default; older entries collapsed in a timeline list, capped at 10 visible with "Load More". |
| 5 | Is the OpenAI Realtime API (WebSocket) a hard requirement for P1, or is the chunked HTTP fallback acceptable for the initial release? | ✅ Answered | HTTP chunked fallback is acceptable for v1. OpenAI WebSocket (Realtime API) is a future enhancement, not a P1 requirement. |
| 6 | What is the maximum expected audio recording duration per session? | ✅ Answered | 5 minutes maximum. Rationale: 5 min of speech-quality audio ≈ 9.5MB — well within OpenAI Whisper's 25MB hard limit, keeps storage costs low, and covers the vast majority of enterprise voice note use cases. |

---

## Change Log

| Date | Author | Change |
|------|--------|---------|
| 2026-04-14 | TPO | Initial specification created from `Salesforce Voice.md` design document |
| 2026-04-14 | TPO | Clarification report executed (Interactive Mode) — 6 spec clarifications + 8 gap findings resolved. Status promoted to Clarified. |
