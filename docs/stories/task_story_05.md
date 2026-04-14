# 003-05 — Cross-Object Compatibility & Mobile Support [US-5]

**Feature**: Salesforce Voice Intelligence Layer ([spec.md](./spec.md))
**Plan**: [plan.md](./plan.md)
**Constitution**: [constitution.md](../../../.sfspeckit/memory/constitution.md)
**Type**: DECLARATIVE
**Priority**: P3

---

## Status

- **State**: READY
- **Assigned To**: Sumanth Yanamala
- **Jira**: [VOICE-05]
- **Branch**: `story/003-05-cross-object-mobile`
- **Started**: —
- **Completed**: —
- **Scores**: —

---

## Description

**As a** Salesforce Administrator  
**I want to** configure both Voice Intelligence components on any standard or custom object record page and verify they work correctly on Salesforce Mobile  
**So that** the feature can be deployed org-wide without requiring per-object code changes, increasing adoption across all teams.

---

## Acceptance Criteria

1. **Given** `voiceNoteRecorder` and `voiceIntelligencePanel` are added to a Case record page layout via the Lightning App Builder, **When** a user navigates to a Case record, **Then** both components render correctly and `@api recordId` is correctly populated with the Case Id.
2. **Given** both components are added to a custom object record page, **When** a user records a voice note on the custom object record, **Then** the ContentDocumentLink is created with `LinkedEntityId = customObjectRecordId` and all pipeline behaviors function identically to Account/Opportunity.
3. **Given** the component `targets` metadata includes `lightning__RecordPage` with `supportedFormFactors: [Small, Large]`, **When** the Salesforce Mobile App is used to navigate to a record, **Then** both components render without layout overflow or broken SLDS styles on a 375px-wide Mobile viewport.
4. **Given** a user on Salesforce Mobile (iOS Safari) clicks "Start Recording", **When** the browser requests microphone access, **Then** the MediaRecorder API initializes (or displays a "Browser not supported" message if the mobile browser does not support MediaRecorder).
5. **Given** a mobile recording is completed and saved, **When** the user returns to the record in `voiceIntelligencePanel`, **Then** the transcript and extracted tasks are visible with correct styling on mobile viewport.

---

## Security & Access Matrix

| Permission Set / Profile | Object | Field(s) | Access (R/E) | Rationale |
|--------------------------|--------|----------|--------------|-----------| 
| `Voice_Intelligence_User` | Any Object (SObject generic) | (none — no new fields on source objects) | N/A | Components use `recordId` only; no source object FLS required |

---

## Test Cases

### Positive Tests ✅
- Add components to Case page layout → both render on Case detail page with correct recordId
- Add to custom object (`Smart_Grid_User_Pref__c`) page → recording creates ContentDocumentLink with correct LinkedEntityId
- `js-meta.xml` specifies `Small` and `Large` form factors → Lightning App Builder allows placing on Mobile-enabled pages
- Mobile viewport (375px): components respond, buttons accessible, transcript area scrollable

### Negative Tests ❌
- Mobile browser without MediaRecorder support → "Browser not supported" message displayed, Stop button hidden
- Non-Voice_Intelligence_User on Mobile attempts recording → PS gate blocks Apex call; LWC surfaces error toast

### Bulk Tests 📊
- N/A — this story is configuration and compatibility validation, not code logic. No bulk scenarios.

---

## Dependencies

- **REQUIRES**: `task_story_00.md` (Foundation)
- **REQUIRES**: `task_story_01.md` (003-01 — `voiceNoteRecorder` must exist)
- **REQUIRES**: `task_story_04.md` (003-04 — `voiceIntelligencePanel` must exist)
- **INDEPENDENT OF**: `task_story_02.md`, `task_story_03.md`

---

## SF Implementation Layers

| Layer | What to Build | SF Skill | File Path | Status |
|-------|-------------|----------|-----------|--------|
| LWC Config | Update `voiceNoteRecorder.js-meta.xml` — ensure `supportedFormFactors` includes `Small` | sf-lwc | `force-app/main/default/lwc/voiceNoteRecorder/voiceNoteRecorder.js-meta.xml` | [ ] |
| LWC Config | Update `voiceIntelligencePanel.js-meta.xml` — ensure `supportedFormFactors` includes `Small` | sf-lwc | `force-app/main/default/lwc/voiceIntelligencePanel/voiceIntelligencePanel.js-meta.xml` | [ ] |
| LWC | Add MediaRecorder browser support detection in `voiceNoteRecorder.js` — show fallback message if unsupported | sf-lwc | `force-app/main/default/lwc/voiceNoteRecorder/voiceNoteRecorder.js` | [ ] |
| LWC CSS | Validate mobile responsive CSS in both components — test at 375px, 768px, 1024px | sf-lwc | Both component `.css` files | [ ] |
| Tests | Manual cross-object test on Case and custom object in sandbox | sf-testing | Manual test — no code file | [ ] |

---

## Scoring Gates

| Layer | SF Skill | Min Score | Target | Actual |
|-------|----------|-----------|--------|--------|
| LWC (config + mobile CSS) | sf-lwc | 125/165 | 140/165 | — |
| Coverage | sf-testing | 90% | 90% | — |

---

## Estimation (Human Developer Effort)

| Layer | Complexity | Estimated Effort (Hours) |
|-------|-----------|--------------------------| 
| LWC — js-meta.xml form factor updates | Low | 0.5 |
| LWC — MediaRecorder browser detection + fallback | Low | 1.0 |
| LWC — Mobile CSS responsive review + fixes | Medium | 1.5 |
| Manual sandbox testing (Case + custom object + Mobile) | Low | 2.0 |
| **Total** | | **5 hours** |

> [!NOTE]
> All estimations are based on **manual developer hours** required to implement and verify the story.

---

## Developer Notes

- `js-meta.xml` `targets` should include:
  ```xml
  <targets>
      <target>lightning__RecordPage</target>
  </targets>
  <targetConfigs>
      <targetConfig targets="lightning__RecordPage">
          <supportedFormFactors>
              <supportedFormFactor type="Small"/>
              <supportedFormFactor type="Large"/>
          </supportedFormFactors>
      </targetConfig>
  </targetConfigs>
  ```
- MediaRecorder detection: `if (!window.MediaRecorder || !MediaRecorder.isTypeSupported('audio/webm')) { this.browserSupported = false; }` — bind to a reactive property to show/hide the recorder UI.
- Mobile CSS: avoid fixed pixel widths; use `max-width: 100%` on all containers. Test transcript area scrollability on mobile (add `overflow-y: auto; max-height: 200px`).
- No new Apex is needed for this story — purely LWC config and CSS adjustments.
- Cross-object compatibility is inherent in the design (`@api recordId` + ContentDocumentLink `LinkedEntityId`). This story validates it via manual testing, not new code.

---

## Code Review Checklist

- [ ] LWC: `js-meta.xml` includes both `Small` and `Large` form factors
- [ ] LWC: MediaRecorder browser detection implemented with user-friendly fallback
- [ ] LWC: No fixed-width CSS that would break on 375px viewport
- [ ] LWC: SLDS 2 responsive utilities used (`slds-size_*` where needed)
- [ ] Tests: Manual testing completed on Case and one custom object
- [ ] Tests: Mobile browser test documented (iOS Safari / Android Chrome)
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
