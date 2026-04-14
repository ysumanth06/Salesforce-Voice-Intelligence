# 003-04 — Voice Intelligence Panel Dashboard [US-4]

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: FULL
**Priority**: P2

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-04]
- **Branch**: `story/003-04-intelligence-panel`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce user on any record page  
**I want to** see a unified dashboard showing all voice notes, their live/final transcripts, extracted tasks, and pipeline status  
**So that** I have complete visibility into my voice intelligence workflow without navigating away from the record or manually checking Tasks and Files separately.

---

## Acceptance Criteria

1. **Given** `voiceIntelligencePanel` is placed on any record page layout, **When** the page loads, **Then** the component calls `AudioRecorderController.getVoiceNotes(recordId)` and displays all ContentVersions with `[Voice]` prefix linked to the record, ordered by `CreatedDate DESC`, with the most recent expanded.
2. **Given** more than 10 voice notes exist on the record, **When** the panel renders, **Then** only the first 10 are shown with a "Load More" button; clicking it loads the next 10.
3. **Given** a voice note's `Transcription_Status__c` is `Completed`, **When** the transcript section renders, **Then** the full text from `Transcript__c` is displayed in a formatted read-only text area; the edit icon is only visible if `currentUserId === ownerId`.
4. **Given** the logged-in user is the ContentVersion owner and they click the edit icon, **When** they modify the transcript text and click Save, **Then** `AudioRecorderController.saveTranscript(contentVersionId, editedText)` is called and `Transcript__c` is updated.
5. **Given** a voice note has `Transcription_Status__c` in progress (Recording, Streaming, Finalizing, Extracting Tasks), **When** the status section renders, **Then** a visual pipeline progress bar shows the current stage highlighted, with "Processing…" animation.
6. **Given** Tasks linked to the source record via `WhatId` have been created, **When** the Tasks section renders, **Then** a table shows Subject, Priority, Due Date, and Status columns; clicking any row opens the Task record in a new tab.
7. **Given** the Queueable job updates `Transcription_Status__c` to `Completed` in the background, **When** the component polls every 5 seconds while status is not terminal, **Then** the UI reflects the updated status without a full page reload.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Field(s) | Access (R/E) | Rationale |
|--------------------------|--------|----------|--------------|-----------| 
| `Voice_Intelligence_User` | ContentVersion | `Transcript__c`, `Partial_Transcript__c`, `Transcription_Status__c`, `Error_Message__c` | Read | Display in panel |
| `Voice_Intelligence_User` | ContentVersion | `Transcript__c` | Edit | Owner can correct transcript |
| `Voice_Intelligence_User` | Task | Subject, Priority, ActivityDate, Status, WhatId | Read | Display in Tasks section |
| `Voice_Intelligence_User` | `AudioRecorderController` | Apex class | Enabled | `getVoiceNotes` and `saveTranscript` calls |

---

## Test Cases

### Positive Tests ✅
- `getVoiceNotes(recordId)` with 3 linked ContentVersions → returns 3 records ordered by CreatedDate DESC
- `getVoiceNotes(recordId)` with 15 linked records → first call returns 10; second call (offset 10) returns 5
- `saveTranscript(cvId, updatedText)` called by owner → `Transcript__c` updated, no exception
- Panel renders pipeline badge for `Finalizing` status → stage 3 of 5 highlighted in progress bar
- Task table renders 3 tasks with correct Subject/Priority/ActivityDate values
- Polling: `Transcription_Status__c` changes from `Extracting Tasks` to `Completed` → UI updates within 1 polling cycle (5s)

### Negative Tests ❌
- `saveTranscript` called by non-owner user → `AuraHandledException`: "Only the recording user may edit the transcript"
- `getVoiceNotes` with invalid recordId → returns empty list (no exception)
- Panel renders with `Status = 'Failed'` → error message from `Error_Message__c` displayed in red banner, no tasks table shown

### Bulk Tests 📊
- Record with 50 linked voice notes → `getVoiceNotes` respects pagination (first 10 returned), no SOQL limit issues
- Task section with 50 Tasks (WhatId = recordId) → table renders with correct pagination (SOQL query returns first 50 Tasks ordered by CreatedDate)

---

## Dependencies

- **REQUIRES**: `task_story_00.md` (Foundation)
- **REQUIRES**: `task_story_01.md` (003-01 — `AudioRecorderController.getVoiceNotes` and `saveTranscript` are added to the controller built in Story-01)
- **REQUIRES**: `task_story_03.md` (003-03 — `VoiceTranscriptionService` must exist for end-to-end panel testing)
- **INDEPENDENT OF**: `task_story_02.md` (streaming controller is backend-only; panel doesn't call it)

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| Apex | Add `getVoiceNotes(recordId, offset)` and `saveTranscript(cvId, text)` methods to `AudioRecorderController` | sf-apex | `force-app/main/default/classes/AudioRecorderController.cls` | [ ] |
| Tests | Add test methods for `getVoiceNotes` and `saveTranscript` to `AudioRecorderControllerTest` | sf-testing | `force-app/main/default/classes/AudioRecorderControllerTest.cls` | [ ] |
| LWC | `voiceIntelligencePanel` — 3-section dashboard (transcript, tasks, status) with pagination and polling | sf-lwc | `force-app/main/default/lwc/voiceIntelligencePanel/` | [ ] |
| LWC Tests | `voiceIntelligencePanel.test.js` — Jest unit tests | sf-testing | `force-app/main/default/lwc/voiceIntelligencePanel/__tests__/voiceIntelligencePanel.test.js` | [ ] |

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| Apex (additions) | sf-apex | 90/150 | 120/150 | — |
| LWC | sf-lwc | 125/165 | 145/165 | — |
| Coverage | sf-testing | 90% | 95% | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| Apex — `getVoiceNotes` + `saveTranscript` additions | Low | 1.5 |
| Apex tests additions | Low | 1.0 |
| LWC — `voiceIntelligencePanel` (3 sections, pagination, polling, edit guard) | Medium | 5.0 |
| LWC — Jest tests | Medium | 2.0 |
| **Total** | | **9.5 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- `getVoiceNotes(Id recordId, Integer offsetCount)`: SOQL queries ContentDocumentLink by `LinkedEntityId = recordId`, joins to ContentVersion where `Title LIKE '[Voice]%'`, orders by `ContentVersion.CreatedDate DESC`, limits 10 with offset. Use `WITH USER_MODE`.
- `saveTranscript(Id contentVersionId, String transcriptText)`: query ContentVersion, check `OwnerId == UserInfo.getUserId()`, if not owner → throw `AuraHandledException`. If owner → update `Transcript__c`. Use `WITH USER_MODE`.
- LWC polling: use `setInterval` (5000ms) only when `Transcription_Status__c` is a non-terminal value (any value other than `Completed`, `Failed`). Clear interval on terminal state or component disconnect.
- Owner check in LWC: retrieve `currentUserId` via `@salesforce/user/Id` import. Compare to `voiceNote.OwnerId` to conditionally show/hide edit button.
- Pipeline status stages map: `Recording=1`, `Streaming=2`, `Finalizing=3`, `Extracting Tasks=4`, `Completed=5`, `Failed=error`, `Queued — Pending Capacity=waiting`.
- Task SOQL: `SELECT Id, Subject, Priority, ActivityDate, Status FROM Task WHERE WhatId = :recordId ORDER BY CreatedDate DESC LIMIT 50 WITH USER_MODE`. Keep in `AudioRecorderController` as a separate `@AuraEnabled` method (`getRelatedTasks(recordId)`).
- Panel layout: use SLDS Accordion for each voice note (most recent open, rest closed). Each accordion section shows transcript + task table + status badge.

---

## Code Review Checklist

- [ ] Apex: Bulkification verified
- [ ] Apex: `with sharing` used
- [ ] Apex: `WITH USER_MODE` on all SOQL
- [ ] Apex: Owner check enforced before transcript edit DML
- [ ] LWC: SLDS 2 compliant — Accordion, Data Table, Progress Indicator used
- [ ] LWC: Polling interval cleared in `disconnectedCallback`
- [ ] LWC: Edit button conditionally rendered (not just hidden) for non-owners
- [ ] Tests: PNB pattern
- [ ] Tests: Coverage ≥ 90%
- [ ] Tests: `VoiceTestDataFactory` used
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
