# 003-06 — Integration Validation & DocGen Conflict Check

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: DECLARATIVE
**Priority**: P1

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-06]
- **Branch**: `story/003-06-integration-validation`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce Developer finalizing the Voice Intelligence Layer deployment  
**I want to** run a full end-to-end integration test across all deployed stories and explicitly verify that the DocGen package does not conflict with voice note ContentVersions  
**So that** the feature is certified safe for production deployment and there are no silent regressions from the DocGen package automation.

---

## Acceptance Criteria

1. **Given** all Stories 003-00 through 003-05 are deployed to the sandbox, **When** a user records a 30-second voice note on an Account record, **Then** the full pipeline completes — ContentVersion created → `VoiceTranscriptionService` executes → `Transcript__c` populated → Tasks created → `Transcription_Status__c = 'Completed'` — within 60 seconds.
2. **Given** the DocGen package is installed in the sandbox, **When** a `[Voice]` prefixed ContentVersion is inserted (voice note saved), **Then** no DocGen automation fires for that file (confirm by checking DocGen job logs or equivalent audit trail — no unexpected document generation for voice files).
3. **Given** DocGen automations exist on ContentVersion (check Flow/Process triggers in sandbox), **When** filtering criteria are reviewed, **Then** either (a) they already exclude `[Voice]` prefixed titles, or (b) an admin updates the DocGen automation filter to exclude `Title LIKE '[Voice]%'` — document this action.
4. **Given** a 5-minute voice note is recorded (maximum duration), **When** the Whisper transcription completes, **Then** the transcript is returned without a "file too large" error from OpenAI (confirms the 10MB gate and 5-min limit work correctly end-to-end).
5. **Given** all scoring gates from Stories 003-00 through 003-05 were met, **When** this story is marked DONE, **Then** the feature is considered production-ready and `/sfspeckit-deploy` may be run.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Access | Rationale |
|--------------------------|--------|--------|-----------|
| N/A | N/A | N/A | This is a validation story — no new permissions introduced |

---

## Test Cases

### Positive Tests ✅
- End-to-end recording on Account → full pipeline completes → `Status = Completed`, Tasks visible in `voiceIntelligencePanel`
- DocGen conflict check: insert `[Voice]` ContentVersion → DocGen job log shows no new document generated for this file
- 5-minute recording (simulated) → Whisper accepts file → transcript returned → Tasks created

### Negative Tests ❌
- DocGen conflict found → document the finding, apply filter fix, re-test. Story is NOT complete until DocGen conflict is resolved or confirmed non-existent.
- Pipeline fails mid-way (simulate API timeout) → `Status = Failed`, `Error_Message__c` populated, no Tasks created, no silent data loss

### Bulk Tests 📊
- 10 voice recordings on the same record simultaneously → all 10 Queueable jobs complete → 10 ContentVersions with `Status = Completed` → `voiceIntelligencePanel` shows all 10 with pagination

---

## Dependencies

- **REQUIRES**: ALL previous stories (003-00 through 003-05) — this is the final integration gate
- **This story BLOCKS**: `/sfspeckit-deploy` — production deployment cannot proceed until this story is DONE

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| Validation | End-to-end manual test on sandbox (Account record) | sf-testing | Manual — no code file | [ ] |
| Validation | DocGen conflict audit — check SF flows/processes triggered on ContentVersion | sf-debug | Manual — run: `sf metadata list -m Flow --target-org sandbox-org` | [ ] |
| Documentation | If DocGen filter update required — document admin action taken in this story file | N/A | Update this file's QA Results section | [ ] |
| Validation | Run all Apex tests: `sf apex run test --target-org sandbox-org --code-coverage` | sf-testing | CLI command | [ ] |

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| All Apex test classes (aggregate) | sf-testing | 90% coverage | 95% | — |
| End-to-end pipeline | Manual | Pass / Fail | Pass | — |
| DocGen conflict | Manual | No conflict / Resolved | No conflict | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| End-to-end sandbox test | Low | 1.0 |
| DocGen audit + filter fix (if needed) | Low-Med | 1.0 |
| Run full Apex test suite + coverage review | Low | 0.5 |
| **Total** | | **2.5 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- To check DocGen automation on ContentVersion:
  ```bash
  sf metadata list -m Flow --target-org sandbox-org
  sf metadata list -m WorkflowRule --target-org sandbox-org
  ```
  Look for any Flow/WFRule that fires on `ContentVersion` object. Review their entry criteria.
- If a DocGen flow fires on ContentVersion insert → add filter condition: `Title Does Not Contain '[Voice]'` to its entry criteria. This is an admin Setup action, not a code change.
- End-to-end test steps:
  1. Navigate to any Account in sandbox
  2. Add `voiceNoteRecorder` and `voiceIntelligencePanel` to the page layout (if not already done)
  3. Record a 10-second voice note
  4. Confirm ContentVersion created with `[Voice]` prefix title
  5. Wait up to 60 seconds for Queueable execution
  6. Confirm `Transcription_Status__c = Completed`, `Transcript__c` populated
  7. Confirm Tasks created in `voiceIntelligencePanel`
  8. Confirm DocGen did NOT generate a document for this voice file
- If any step fails → file a bug, fix in the relevant story branch, re-run this validation.

---

## Code Review Checklist

- [ ] All Apex test classes pass with ≥ 90% coverage
- [ ] No open bugs from Stories 003-01 through 003-05
- [ ] DocGen conflict confirmed absent or resolved and documented
- [ ] End-to-end pipeline tested and passing
- [ ] All story branches merged to `feature/003-salesforce-voice-intelligence-layer`

**Peer Reviewer**: Sumanth Yanamala — [ ] Approved  
**Architect**: Sumanth Yanamala — [ ] Approved

---

## QA Results

- **Test Scripts Generated**: —
- **Automated Tests**: —/— passed
- **Manual Tests**: —/— passed
- **Coverage**: —%
- **DocGen Conflict**: [ ] None Found / [ ] Found & Resolved (describe below)
- **DocGen Resolution Notes**: —
- **QA Verdict**: [ ] PASS / [ ] FAIL
- **QA Tester**: —
- **Date**: —
