# 003-03 — Voice Transcription Service & Task Extraction [US-3]

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: FULL
**Priority**: P1

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-03]
- **Branch**: `story/003-03-transcription-task-extraction`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce user who has completed a voice recording  
**I want the** system to automatically transcribe the full recording and extract any action items as Salesforce Tasks  
**So that** follow-up commitments from conversations are captured in the CRM without manual data entry.

---

## Acceptance Criteria

1. **Given** a ContentVersion with `Transcription_Status__c = 'Finalizing'` is passed to `VoiceTranscriptionService`, **When** the Queueable executes, **Then** a Whisper API callout is made, the full transcript is saved to `Transcript__c`, and status is updated to `Extracting Tasks`.
2. **Given** the Whisper transcription succeeds, **When** the GPT-4.1-mini task extraction callout is made, **Then** the prompt includes the full transcript and requests a strict JSON array response.
3. **Given** GPT-4.1-mini returns valid JSON, **When** Apex parses the response, **Then** up to 10 Task records are created with `WhatId` = source record Id, `Subject`, `Description`, `Priority`, `ActivityDate` (or Today+7 if null), `OwnerId` = ContentVersion owner.
4. **Given** GPT-4.1-mini returns invalid JSON on the first attempt, **When** Apex retries, **Then** a second GPT-4.1-mini call is made with the same prompt; if still invalid, GPT-4.1 is called as fallback.
5. **Given** all 3 LLM calls fail or return invalid JSON, **When** the error state is reached, **Then** `Transcription_Status__c` is set to `Failed`, `Error_Message__c` is populated with detail, and no partial Tasks are committed.
6. **Given** the LLM returns an array with more than 10 tasks, **When** Apex processes the list, **Then** only the first 10 are inserted and the remainder are logged in `Error_Message__c` as "Truncated: X additional tasks discarded".
7. **Given** the LLM returns `[]` (no tasks), **When** execution completes, **Then** `Transcription_Status__c` is set to `Completed`, no Tasks are created, and no error is raised.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Field(s) | Access (R/E) | Rationale |
|--------------------------|--------|----------|--------------|-----------| 
| System (Queueable runs in Apex system context) | ContentVersion | All 5 custom fields | Read/Edit | Queueable updates status and transcript |
| System | Task | Subject, Description, Priority, ActivityDate, WhatId, OwnerId | Create | Auto-created from extraction |
| System | ContentDocumentLink | LinkedEntityId | Read | Needed to determine WhatId (source record) |

> **Note**: `VoiceTranscriptionService` runs in Queueable context. It must use `with sharing` (Constitution IV) but will execute as the enqueuing user — the recording user. Validate that the recording user has Task create permission (provided by `Voice_Intelligence_User` PS).

---

## Test Cases

### Positive Tests ✅
- `VoiceTranscriptionService` with valid ContentVersion Id → Whisper mock returns transcript → GPT mock returns 2-task JSON → 2 Tasks created, `Status = 'Completed'`
- Task with null `dueDate` from LLM → `ActivityDate` = `Date.today().addDays(7)`
- Task with `priority = 'High'` → Salesforce Task `Priority = 'High'` correctly mapped
- LLM returns array of 11 tasks → only 10 Tasks inserted, `Error_Message__c` contains truncation message

### Negative Tests ❌
- GPT-4.1-mini returns invalid JSON → retry → GPT-4.1-mini still invalid → fallback to GPT-4.1 → GPT-4.1 returns valid JSON → Tasks created (3-callout chain test)
- All 3 LLM calls return invalid JSON → `Status = 'Failed'`, `Error_Message__c` set, no Tasks created (atomic: no partial commit)
- Whisper callout returns HTTP 500 → `Status = 'Failed'`, `Error_Message__c` = "Transcription failed: HTTP 500"
- LLM returns `[]` → no Tasks created, `Status = 'Completed'`
- `ContentVersion` Id not found (deleted between enqueue and execute) → `QueryException` caught, `Status` not updated (record gone), error logged

### Bulk Tests 📊
- 50 Queueable executions in test context (1 per ContentVersion) → each creates up to 10 Tasks → 500 total Task inserts — verify no DML limits hit (each Queueable is a separate transaction)
- Single Queueable with 10-task extraction → DML called once for all 10 Tasks (bulkified `insert tasks`)

---

## Dependencies

- **REQUIRES**: `task_story_00.md` (Foundation — Named Credential, ContentVersion fields, VoiceTestDataFactory)
- **REQUIRES**: `task_story_01.md` (003-01 — `AudioRecorderController` enqueues this service; must be deployed before integration testing)
- **INDEPENDENT OF**: `task_story_02.md`

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| Apex | `VoiceTranscriptionService` — Queueable, Whisper callout, GPT pipeline, retry logic, Task creation | sf-apex | `force-app/main/default/classes/VoiceTranscriptionService.cls` | [ ] |
| Apex | Meta XML | sf-apex | `force-app/main/default/classes/VoiceTranscriptionService.cls-meta.xml` | [ ] |
| Tests | `VoiceTranscriptionServiceTest` — PNB with multi-step `HttpCalloutMock` | sf-testing | `force-app/main/default/classes/VoiceTranscriptionServiceTest.cls` | [ ] |

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| Apex | sf-apex | 90/150 | 125/150 | — |
| Coverage | sf-testing | 90% | 95% | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| Apex — `VoiceTranscriptionService` (Whisper + GPT pipeline + retry) | High | 6.0 |
| Apex — `VoiceTranscriptionServiceTest` (multi-mock chain) | High | 4.0 |
| **Total** | | **10 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- **Callout budget per Queueable**: Whisper (1) + GPT attempt 1 (1) + GPT attempt 2/fallback (1) = max 3 callouts. Well within the 100 callout limit.
- **Multi-step mock**: Use `MultiRequestMock` pattern (implement `HttpCalloutMock` with a queue of responses). Test the full 3-callout chain in a single test method.
- **JSON validation**: use `JSON.deserializeUntyped()` first, then cast and validate field presence. If `subject` key is missing → invalid.
- **Atomic Task creation**: build the full `List<Task>` before any DML. If JSON parsing fails at any point, throw and catch — zero DML executed.
- **WhatId resolution**: query `ContentDocumentLink` where `ContentDocumentId` = ContentVersion's `ContentDocumentId` and `LinkedEntityId != null AND LinkedEntityId != ContentDocumentId` to get the source record Id.
- **Task insert**: single `insert tasks;` call (bulkified). Never insert in a loop.
- **GPT prompt**: system role = "You are a system that extracts actionable tasks from transcripts. Return ONLY valid JSON array. If no tasks, return []." User role = transcript text.
- **Model names**: primary = `gpt-4.1-mini`, fallback = `gpt-4.1`. Pass as `"model"` in the chat completions JSON body.
- All status updates must be a single `update cv;` call at the end — do not update status mid-execution to avoid partial state.

---

## Code Review Checklist

- [ ] Apex: Bulkification verified — Task insert is a single DML call on a list
- [ ] Apex: `with sharing` used
- [ ] Apex: `WITH USER_MODE` on all SOQL (ContentVersion query, CDL query)
- [ ] Apex: No hardcoded IDs
- [ ] Apex: `HttpCalloutMock` used for all test methods (`Test.setMock`)
- [ ] Tests: 3-callout chain tested (primary → retry → fallback)
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
