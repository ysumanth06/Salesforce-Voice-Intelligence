# 003-01 — Audio Recording & File Save [US-1]

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: FULL
**Priority**: P1

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-01]
- **Branch**: `story/003-01-audio-recording-file-save`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce user (Sales Rep, Service Agent, Account Manager)  
**I want to** start and stop a voice recording directly on any Salesforce record page and have the audio automatically saved  
**So that** I can capture conversation context without leaving Salesforce, with the recording securely stored as a file linked to the record.

---

## Acceptance Criteria

1. **Given** the `voiceNoteRecorder` LWC is on a record page and the user has `Voice_Intelligence_User` PS, **When** they click "Start Recording", **Then** the browser requests microphone permission and begins capturing audio via the `MediaRecorder` API with `timeslice(1000)` chunks.
2. **Given** a recording is in progress, **When** the user clicks "Stop Recording", **Then** `AudioRecorderController.saveAudioChunk` is called with the complete base64-encoded audio; a ContentVersion is inserted with `Title = '[Voice] <RecordName> <timestamp>'` and linked to the source record via a ContentDocumentLink.
3. **Given** the audio file submission exceeds 10MB, **When** `AudioRecorderController` receives the file, **Then** it throws an `AuraHandledException` with message "Recording exceeds maximum size. Please keep recordings under 5 minutes." and no ContentVersion is created.
4. **Given** the user has no microphone permission, **When** they click "Start Recording", **Then** the LWC catches the `NotAllowedError` and displays an error toast — no recording session is initiated.
5. **Given** a recording is active and the user navigates away from the page, **When** the LWC `disconnectedCallback` fires, **Then** `MediaRecorder.stop()` is called and any buffered audio chunks are discarded gracefully (no partial upload).
6. **Given** a successful file save, **When** `AudioRecorderController` enqueues `VoiceTranscriptionService`, **Then** the `Transcription_Status__c` field is set to `Queued — Pending Capacity` if the flex queue is full, or `Finalizing` if the job was enqueued successfully.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Field(s) | Access (R/E) | Rationale |
|--------------------------|--------|----------|--------------|-----------| 
| `Voice_Intelligence_User` | ContentVersion | `Transcription_Status__c`, `Streaming_Session_Id__c` | Read/Edit | Set on insert by controller |
| `Voice_Intelligence_User` | ContentVersion | `Error_Message__c` | Read | Display errors in UI |
| `Voice_Intelligence_User` | ContentVersion | (base) | Create/Read | Upload audio file |
| `Voice_Intelligence_User` | ContentDocumentLink | (base) | Create/Read | Link file to record |
| `Voice_Intelligence_User` | `AudioRecorderController` | Apex class | Enabled | LWC calls this |

---

## Test Cases

### Positive Tests ✅
- `AudioRecorderController.saveAudioChunk(validBase64, recordId, '[Voice] Test', sessionId)` → ContentVersion inserted, ContentDocumentLink created, `Transcription_Status__c = 'Finalizing'`, `System.enqueueJob` called
- File exactly at 10MB boundary → accepted, ContentVersion created
- User with `Voice_Intelligence_User` PS calls `saveAudioChunk` → no FLS exception, record created
- LWC start/stop recording cycle → MediaRecorder starts, 1-second chunks fire, stop sends full audio

### Negative Tests ❌
- `saveAudioChunk` with file > 10MB → `AuraHandledException` thrown, no ContentVersion created
- `saveAudioChunk` with null recordId → `AuraHandledException` thrown: "Record ID is required"
- User without `Voice_Intelligence_User` PS calls `saveAudioChunk` → `NoAccessException` from FLS enforcement
- `System.enqueueJob` throws `AsyncException` (queue full) → `Transcription_Status__c` set to `Queued — Pending Capacity`, no uncaught exception

### Bulk Tests 📊
- 50 concurrent `saveAudioChunk` calls in test context → all ContentVersions inserted, all Queueables enqueued (verify flex queue limit handling)

---

## Dependencies

- **REQUIRES**: `task_story_00.md` (Foundation — ContentVersion fields, Named Credential, PS, VoiceTestDataFactory)
- **INDEPENDENT OF**: `task_story_02.md`, `task_story_03.md`

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| Apex | `AudioRecorderController` — `saveAudioChunk`, `validateFileSize`, `getVoiceNotes` | sf-apex | `force-app/main/default/classes/AudioRecorderController.cls` | [ ] |
| Apex | `AudioRecorderController` meta | sf-apex | `force-app/main/default/classes/AudioRecorderController.cls-meta.xml` | [ ] |
| Tests | `AudioRecorderControllerTest` — PNB coverage | sf-testing | `force-app/main/default/classes/AudioRecorderControllerTest.cls` | [ ] |
| LWC | `voiceNoteRecorder` — recording UI, timer, permission handling | sf-lwc | `force-app/main/default/lwc/voiceNoteRecorder/` | [ ] |
| LWC Tests | `voiceNoteRecorder.test.js` — Jest unit tests | sf-testing | `force-app/main/default/lwc/voiceNoteRecorder/__tests__/voiceNoteRecorder.test.js` | [ ] |

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| Apex | sf-apex | 90/150 | 120/150 | — |
| LWC | sf-lwc | 125/165 | 140/165 | — |
| Coverage | sf-testing | 90% | 95% | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| Apex — `AudioRecorderController` | Medium | 3.0 |
| Apex — `AudioRecorderControllerTest` | Medium | 2.0 |
| LWC — `voiceNoteRecorder` (MediaRecorder, timer, error handling) | High | 6.0 |
| LWC — Jest tests | Medium | 2.0 |
| **Total** | | **13 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- `AudioRecorderController` must use `with sharing` (Constitution Article IV).
- `saveAudioChunk` is `@AuraEnabled(cacheable=false)` — it performs DML.
- File size validation: `EncodingUtil.base64Decode(base64String).size()` returns byte count. Gate at 10,485,760 bytes (10MB).
- ContentVersion insert pattern: `PathOnClient = '[Voice] Recording.webm'`, `Title = '[Voice] ' + timestamp`, `FirstPublishLocationId = null` (insert CV first, then CDL separately).
- ContentDocumentLink: `LinkedEntityId = recordId`, `ShareType = 'I'` (Inferred), `Visibility = 'AllUsers'`.
- `VoiceTranscriptionService` is enqueued with the ContentVersion Id. Wrap `System.enqueueJob()` in try/catch for `AsyncException` — set status to `Queued — Pending Capacity` on catch.
- LWC: Use `crypto.randomUUID()` or timestamp to generate the `sessionId` passed to Apex.
- LWC 5-minute timer: `setInterval` counting 300 seconds → auto-call stop handler. Show countdown.
- MediaRecorder `mimeType` preference: `'audio/webm;codecs=opus'` (best Whisper compatibility on Chrome/Edge). Fallback: `'audio/ogg'`.

---

## Code Review Checklist

- [ ] Apex: Bulkification verified (no SOQL/DML in loops)
- [ ] Apex: `with sharing` used
- [ ] Apex: `WITH USER_MODE` on all SOQL queries
- [ ] Apex: No hardcoded Salesforce IDs
- [ ] LWC: SLDS 2 compliant
- [ ] LWC: Keyboard accessible (Start/Stop buttons have `aria-label`)
- [ ] Tests: PNB pattern (Positive, Negative, Bulk 251+)
- [ ] Tests: Coverage ≥ 90%
- [ ] Tests: `VoiceTestDataFactory` used
- [ ] Tests: Assert class with descriptive messages
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
