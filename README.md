# Salesforce Voice Intelligence Layer

> **Status**: 📋 Specification Complete — Pending Implementation

A production-ready Salesforce-native Voice Intelligence Layer that records voice notes on any record, streams real-time transcription via OpenAI Whisper, extracts actionable tasks using GPT-4.1-mini, and displays everything in an inline dashboard — all without leaving Salesforce.

---

## 📁 Documentation

| File | Description |
|------|-------------|
| [`docs/requirements.md`](docs/requirements.md) | Original feature requirements and architecture vision |
| [`docs/spec.md`](docs/spec.md) | Functional specification (SFSpeckit) — user stories, FRs, security model |
| [`docs/clarification-report.md`](docs/clarification-report.md) | Clarification report — gap analysis, 10-point checklist, stakeholder sign-off |
| [`docs/plan.md`](docs/plan.md) | Technical implementation plan — architecture, deployment order, scoring gates |
| [`docs/data-model.md`](docs/data-model.md) | Data model — ContentVersion field definitions, ERD, Permission Set, Named Credential |

---

## 🏗️ Architecture Overview

```
voiceNoteRecorder (LWC)
  ├──► AudioRecorderController (Apex)         → Save audio as ContentVersion
  └──► StreamingTranscriptionController (Apex) → Stateless chunk relay to Whisper

VoiceTranscriptionService (Queueable Apex)
  ├── OpenAI Whisper → final transcript
  ├── GPT-4.1-mini → task extraction (JSON)
  └── Task records created (max 10, WhatId = source record)

voiceIntelligencePanel (LWC)
  ├── Live / Final Transcript (editable by owner)
  ├── Extracted Tasks Table (auto-refreshing)
  └── Pipeline Status Badge (Recording → Completed / Failed)
```

---

## 📦 Key Design Decisions

| Decision | Value |
|---------|-------|
| Max recording duration | 5 minutes (UI timer + 10MB server gate) |
| Max tasks per transcript | 10 (Apex-enforced) |
| LLM primary model | GPT-4.1-mini |
| LLM fallback model | GPT-4.1 |
| Transcription model | OpenAI Whisper (batch, post-recording) |
| Auth | Named Credential + External Credential (zero hardcoded keys) |
| Task due date (null) | Today + 7 days |
| File identification | `[Voice]` title prefix on all ContentVersions |

---

## 🔐 Security

- All OpenAI API calls routed through Apex Named Credentials
- Recording access gated by `Voice_Intelligence_User` Permission Set
- Transcript editing restricted to the original recording user (ContentVersion owner)
- `with sharing` + `WITH USER_MODE` enforced on all Apex

---

## 📋 Built With SFSpeckit

This project was specified, clarified, and planned using the **SFSpeckit** AI-Driven Development (AIDD) framework.

[![SFSpeckit](https://img.shields.io/badge/Built%20with-SFSpeckit-blueviolet)](https://github.com/ysumanth06/SFSpeckit)
