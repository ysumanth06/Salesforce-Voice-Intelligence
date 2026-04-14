# 003-00 — Foundation: Metadata, Named Credentials & Permission Set

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: FULL
**Priority**: P1

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-00]
- **Branch**: `story/003-00-foundation`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce Developer  
**I want to** deploy all foundational metadata — ContentVersion custom fields, Named Credentials, External Credentials, Permission Set, and a VoiceTestDataFactory class — before any other story begins  
**So that** all subsequent stories have a stable, pre-existing data model and authentication infrastructure to build against without risk of compilation failures or missing schema references.

---

## Acceptance Criteria

1. **Given** the feature branch is checked out, **When** the 5 ContentVersion custom fields are deployed, **Then** all fields are visible in Setup → Object Manager → ContentVersion → Fields & Relationships with correct types and labels.
2. **Given** the ContentVersion fields are deployed, **When** the `Voice_Intelligence_User` Permission Set is deployed, **Then** all 5 fields appear in the PS's Field-Level Security with Read access set, and `Transcript__c` + `Transcription_Status__c` + `Partial_Transcript__c` + `Streaming_Session_Id__c` set as Editable.
3. **Given** the Permission Set is deployed, **When** a test user is assigned the `Voice_Intelligence_User` PS, **Then** they can read and edit the voice-related ContentVersion fields without errors.
4. **Given** the External Credential and Named Credential are deployed, **When** an Apex anonymous script calls `callout:OpenAI_API/v1/models`, **Then** the call reaches OpenAI (or returns an auth error, not a named credential config error).
5. **Given** the `VoiceTestDataFactory` class is deployed, **When** test classes in Stories 003-01 through 003-06 reference it, **Then** there are no compilation errors and test data is produced correctly.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Field(s) | Access (R/E) | Rationale |
|--------------------------|--------|----------|--------------|-----------| 
| `Voice_Intelligence_User` | ContentVersion | `Transcription_Status__c` | Read/Edit | App users must update pipeline status |
| `Voice_Intelligence_User` | ContentVersion | `Transcript__c` | Read/Edit | Users read final transcript; owners edit |
| `Voice_Intelligence_User` | ContentVersion | `Partial_Transcript__c` | Read/Edit | Streaming partial transcript |
| `Voice_Intelligence_User` | ContentVersion | `Streaming_Session_Id__c` | Read/Edit | Session tracking during recording |
| `Voice_Intelligence_User` | ContentVersion | `Error_Message__c` | Read Only | Users see errors; Apex writes via system context |
| `Voice_Intelligence_User` | ContentVersion | (base object) | Create/Read/Edit | Upload audio files |
| `Voice_Intelligence_User` | ContentDocument | (base object) | Read | Browse linked files |
| `Voice_Intelligence_User` | Task | (base object) | Create/Read/Edit | Create and manage extracted tasks |
| `Voice_Intelligence_User` | `AudioRecorderController` | (Apex class) | Enabled | LWC calls this controller |
| `Voice_Intelligence_User` | `StreamingTranscriptionController` | (Apex class) | Enabled | LWC calls this controller |

---

## Test Cases

### Positive Tests ✅
- Deploy ContentVersion fields → all 5 appear in Object Manager without errors
- Assign `Voice_Intelligence_User` PS to a test user → FLS grants confirmed via SOQL tooling query
- Named Credential deploys → appears in Setup → Named Credentials list
- `VoiceTestDataFactory.createVoiceNote(recordId)` → returns a ContentVersion with `Transcription_Status__c` = `Recording`

### Negative Tests ❌
- Attempt to edit `Error_Message__c` from LWC as a non-owner user → FLS blocks edit (read-only enforced)
- Deploy PS without fields deployed first → deployment fails with missing field reference error (validates deploy order)

### Bulk Tests 📊
- `VoiceTestDataFactory.createVoiceNotes(recordId, 10)` → creates 10 ContentVersion records in a single DML, all fields populated correctly
- No governor limit issues with 10 simultaneous insertions

---

## Dependencies

- **REQUIRES**: Nothing — this is the root story
- **BLOCKS**: ALL other stories (003-01 through 003-06)

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| Metadata | `Transcription_Status__c` picklist field on ContentVersion | sf-metadata | `force-app/main/default/objects/ContentVersion/fields/Transcription_Status__c.field-meta.xml` | [ ] |
| Metadata | `Transcript__c` long text field | sf-metadata | `force-app/main/default/objects/ContentVersion/fields/Transcript__c.field-meta.xml` | [ ] |
| Metadata | `Partial_Transcript__c` long text field | sf-metadata | `force-app/main/default/objects/ContentVersion/fields/Partial_Transcript__c.field-meta.xml` | [ ] |
| Metadata | `Streaming_Session_Id__c` text field | sf-metadata | `force-app/main/default/objects/ContentVersion/fields/Streaming_Session_Id__c.field-meta.xml` | [ ] |
| Metadata | `Error_Message__c` long text field | sf-metadata | `force-app/main/default/objects/ContentVersion/fields/Error_Message__c.field-meta.xml` | [ ] |
| Metadata | `Voice_Intelligence_User` Permission Set | sf-permissions | `force-app/main/default/permissionsets/Voice_Intelligence_User.permissionset-meta.xml` | [ ] |
| Integration | `OpenAI_External_Credential` External Credential | sf-integration | `force-app/main/default/externalCredentials/OpenAI_External_Credential.externalCredential-meta.xml` | [ ] |
| Integration | `OpenAI_API` Named Credential | sf-integration | `force-app/main/default/namedCredentials/OpenAI_API.namedCredential-meta.xml` | [ ] |
| Apex | `VoiceTestDataFactory` test data factory class | sf-apex | `force-app/main/default/classes/VoiceTestDataFactory.cls` | [ ] |
| Tests | `VoiceTestDataFactoryTest` unit test | sf-testing | `force-app/main/default/classes/VoiceTestDataFactoryTest.cls` | [ ] |

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| Metadata (fields) | sf-metadata | 84/120 | 100/120 | — |
| Metadata (PS) | sf-metadata | 84/120 | 100/120 | — |
| Integration (Named Cred) | sf-metadata | 84/120 | 95/120 | — |
| Apex (Factory) | sf-apex | 90/150 | 120/150 | — |
| Coverage | sf-testing | 90% | 95% | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| Metadata (5 fields) | Low | 1.5 |
| Permission Set | Low | 0.5 |
| Named Credential + External Credential | Low | 1.0 |
| VoiceTestDataFactory + Test | Low | 1.5 |
| **Total** | | **4.5 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- Deploy ContentVersion fields **before** the Permission Set — PS deployment references field API names.
- Deploy Named Credential **before** any Apex controller — controllers reference `callout:OpenAI_API` and will fail to compile if the named credential doesn't exist.
- External Credential must have a principal configured with the OpenAI API key **manually in Setup** — this cannot be automated via metadata deployment (API keys are secrets).
- `VoiceTestDataFactory` should follow the same pattern as existing test helpers in the codebase. Create a `createVoiceNote(Id linkedRecordId)` method that inserts a ContentVersion + ContentDocumentLink.
- The `[Voice]` title prefix (FR-025) must be applied in the factory method: `Title = '[Voice] Test Recording'`.

---

## Code Review Checklist

- [ ] Apex: Bulkification verified (no SOQL/DML in loops)
- [ ] Apex: `with sharing` used (or exception documented)
- [ ] Apex: `WITH USER_MODE` on all SOQL queries
- [ ] Apex: No hardcoded Salesforce IDs
- [ ] LWC: N/A — no LWC in this story
- [ ] Tests: PNB pattern (Positive, Negative, Bulk 251+)
- [ ] Tests: Coverage ≥ 90%
- [ ] Tests: Test Data Factory used (no inline data creation)
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
