# 003-02 — Streaming Transcription Controller [US-2]

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: FULL
**Priority**: P1

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-02]
- **Branch**: `story/003-02-streaming-transcription`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce user actively recording a voice note  
**I want to** see chunks of my transcript appear progressively on screen as I speak  
**So that** I can verify the transcription is capturing my words correctly in near real-time and correct any critical errors immediately.

---

## Acceptance Criteria

1. **Given** the `voiceNoteRecorder` LWC is recording, **When** each 1-second audio chunk is produced by `MediaRecorder.ondataavailable`, **Then** the LWC base64-encodes the chunk and calls `StreamingTranscriptionController.transcribeChunk(base64, sessionId)` imperatively.
2. **Given** `transcribeChunk` receives a valid audio chunk, **When** the Whisper callout succeeds, **Then** the partial transcript text is returned to the LWC and appended to the live transcript display area within 3 seconds.
3. **Given** the OpenAI callout returns a non-200 response, **When** the controller processes the error, **Then** it returns an empty string (`''`) — the LWC shows a brief "Processing..." indicator and retries on the next chunk without breaking the recording session.
4. **Given** multiple chunks arrive rapidly (debounce scenario), **When** the LWC receives `ondataavailable` events faster than Apex can respond, **Then** the LWC queues chunks and sends them sequentially, not in parallel, to avoid callout limit exhaustion.
5. **Given** the recording session ends, **When** all streaming chunks have been processed, **Then** the `partialTranscript` LWC state is preserved and displayed as a preview until the final `Transcript__c` from the Queueable job overwrites it.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Field(s) | Access (R/E) | Rationale |
|--------------------------|--------|----------|--------------|-----------| 
| `Voice_Intelligence_User` | ContentVersion | `Partial_Transcript__c` | Read/Edit | Written by controller per chunk (optional) |
| `Voice_Intelligence_User` | `StreamingTranscriptionController` | Apex class | Enabled | LWC calls this controller |

---

## Test Cases

### Positive Tests ✅
- `transcribeChunk(validBase64, sessionId)` → Whisper callout made, partial transcript string returned (non-empty)
- `transcribeChunk` with mock HTTP 200 response → returns transcript text `"Hello, testing."` correctly parsed
- LWC appends 5 sequential chunk responses → live transcript area shows concatenated text in order
- LWC debounce logic → when 3 chunks fire within 500ms, they are processed sequentially (queue verified via Jest spy)

### Negative Tests ❌
- `transcribeChunk` with HTTP 429 (rate limit) → returns `''`, logs error to `Error_Message__c`, LWC shows "Processing..." (no exception thrown to UI)
- `transcribeChunk` with HTTP 500 → returns `''`, no uncaught exception
- `transcribeChunk` with null/empty base64 → `AuraHandledException` thrown: "Audio chunk is required"
- Callout times out (>10s) → `System.CalloutException` caught, returns `''`

### Bulk Tests 📊
- Simulate 300 sequential chunk calls in test (using mock) → all succeed within governor limits (each call is a separate transaction — N/A for limit stacking, but validate mock response parsing at scale)

---

## Dependencies

- **REQUIRES**: `task_story_00.md` (Foundation — Named Credential must exist for callout compilation)
- **INDEPENDENT OF**: `task_story_01.md` (parallel development possible, but LWC integration requires both)

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| Apex | `StreamingTranscriptionController` — `transcribeChunk(base64, sessionId)` | sf-apex | `force-app/main/default/classes/StreamingTranscriptionController.cls` | [ ] |
| Apex | Meta XML | sf-apex | `force-app/main/default/classes/StreamingTranscriptionController.cls-meta.xml` | [ ] |
| Tests | `StreamingTranscriptionControllerTest` — PNB with `HttpCalloutMock` | sf-testing | `force-app/main/default/classes/StreamingTranscriptionControllerTest.cls` | [ ] |

> **Note**: LWC chunk-sending logic is implemented as part of Story 003-01 (`voiceNoteRecorder`). This story delivers the Apex endpoint that the LWC calls.

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| Apex | sf-apex | 90/150 | 120/150 | — |
| Coverage | sf-testing | 90% | 95% | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| Apex — `StreamingTranscriptionController` | Medium | 2.5 |
| Apex — `StreamingTranscriptionControllerTest` (with HttpCalloutMock) | Medium | 2.0 |
| **Total** | | **4.5 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- `StreamingTranscriptionController` is **stateless** — each call processes one chunk independently. No instance variables.
- Callout pattern: `req.setEndpoint('callout:OpenAI_API/v1/audio/transcriptions')`, `req.setMethod('POST')`.
- Whisper requires multipart form data. Build the body manually in Apex using a boundary string (standard Salesforce pattern for multipart callouts).
- Audio format: pass `model = 'whisper-1'` and `response_format = 'text'` to get a plain string back (not JSON) — simplifies parsing.
- The method signature: `@AuraEnabled public static String transcribeChunk(String base64Audio, String sessionId)`.
- Return `''` (empty string) on any error rather than throwing — the LWC is resilient to empty partial transcripts.
- `HttpCalloutMock` must be used in all test methods — no real callouts in tests. Create `WhisperMockResponse` inner class in the test file.
- Validate: this class must be callable from both Story-01 LWC (`voiceNoteRecorder`) context — ensure it's in the `Voice_Intelligence_User` PS Apex class access list (done in Story-00).

---

## Code Review Checklist

- [ ] Apex: `with sharing` used
- [ ] Apex: `WITH USER_MODE` on SOQL (no SOQL needed in this class — controller is callout-only)
- [ ] Apex: No hardcoded Salesforce IDs
- [ ] Apex: `HttpCalloutMock` implemented for all test methods
- [ ] Tests: PNB pattern
- [ ] Tests: Coverage ≥ 90%
- [ ] Tests: `VoiceTestDataFactory` used where applicable
- [ ] Constitution: No violations

**Peer Reviewer**: Sumanth Yanamala — [ ] Approved  
**Architect**: Sumanth Yanamala — [ ] Approved

---

## QA Results

- **Test Scripts Generated**: —
- **Automated Tests**: —
- **Manual Tests**: —
- **Coverage**: —
- **QA Verdict**: [ ] PASS / [ ] FAIL
- **QA Tester**: —
- **Date**: —
