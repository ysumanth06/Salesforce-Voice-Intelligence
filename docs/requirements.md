You are a senior Salesforce Architect and Full-Stack Engineer.

Build a COMPLETE, production-ready Salesforce solution for:

- Voice recording on any record
- REAL-TIME streaming transcription
- AI-based task extraction
- Inline UI dashboard showing transcript + tasks
- Automatic Task creation

This must be a minimal-stack, enterprise-grade, reusable architecture.

---

## 🎯 CORE GOAL

Create a Salesforce-native "Voice Intelligence Layer" that:

1. Records voice notes
2. Streams transcription in real-time (NOT batch)
3. Extracts tasks using AI
4. Displays transcript + tasks directly on record page
5. Works across ALL objects (standard + custom)

---

## 🏗️ HIGH-LEVEL ARCHITECTURE

1. LWC: Voice Recorder + Live Transcript Viewer
2. Apex: File handling + streaming bridge + orchestration
3. Streaming Service (OpenAI Realtime API OR chunked streaming simulation)
4. Queueable Apex fallback for final transcription
5. Flow: task creation + orchestration fallback
6. UI component: Transcript + Task panel (embedded on record page)

---

## ⚡ KEY NEW FEATURE #1: REAL-TIME STREAMING TRANSCRIPTION

Instead of batch Whisper processing, implement:

OPTION A (PREFERRED - REAL STREAMING):

- Use OpenAI Realtime API (WebSocket)
- Stream audio chunks from browser → API
- Receive partial transcripts in real time

OPTION B (FALLBACK - SIMULATED STREAMING):

- Split audio into 1–3 second chunks in LWC
- Send chunks sequentially via Apex
- Append partial transcripts progressively

---

## 📦 STREAMING REQUIREMENTS (LWC)

Enhance LWC (voiceNoteRecorder) to:

- Capture audio in chunks using MediaRecorder timeslice API:
  MediaRecorder.start(1000) // 1-second chunks

- Stream chunks immediately to Apex or WebSocket bridge

- Display:
  - Live transcript area (updates every few seconds)
  - Recording status
  - Latency indicator ("Processing...")

- Maintain:
  - partialTranscript state
  - finalTranscript state

---

## 📡 APEX STREAMING CONTROLLER

Create:

StreamingTranscriptionController

Responsibilities:

- Accept audio chunks (base64)
- Forward to streaming API OR buffer
- Return partial transcript updates

Must support:

- near real-time response
- stateless processing per chunk

---

## 🧠 STREAMING AI DESIGN

If OpenAI Realtime API is used:

- Maintain session per recording
- Stream audio → receive partial JSON:
  {
  "type": "transcript.delta",
  "text": "..."
  }

If not available:

- fallback to chunk aggregation + Whisper batch at end

---

## ⚡ KEY NEW FEATURE #2: INLINE UI DASHBOARD

Create a Lightning Web Component:

voiceIntelligencePanel

This must be placed on ANY record page and display:

---

## 📊 SECTION 1: LIVE TRANSCRIPT

- Real-time scrolling transcript
- Editable text area (user can correct text)
- Auto-refresh every few seconds (via LDS or polling Apex)

Fields source:

- Transcript\_\_c (ContentVersion or custom object)

UI features:

- Highlight new incoming text
- "Finalizing..." indicator

---

## 📋 SECTION 2: EXTRACTED TASKS

Display tasks in a table:

Columns:

- Subject
- Description
- Priority
- Due Date
- Status (Created / Pending)

Source:

- Task object filtered by WhatId = recordId

Features:

- Auto-refresh (Lightning Data Service or wire adapters)
- Click task → open record

---

## 🔄 SECTION 3: PROCESS STATUS

Show pipeline status:

- Recording
- Streaming transcription
- Finalizing transcript
- Extracting tasks
- Completed / Failed

Source:
ContentVersion.Transcription_Status\_\_c

---

## 📦 COMPONENT BREAKDOWN

### 1. voiceNoteRecorder (LWC)

- Records audio
- Streams chunks (NEW)
- Shows live transcript preview (NEW)

### 2. voiceIntelligencePanel (NEW LWC)

- Displays:
  - Transcript (live + final)
  - Tasks list
  - Status pipeline

### 3. AudioRecorderController (Apex)

- Save file
- Initialize streaming session

### 4. StreamingTranscriptionController (NEW Apex)

- Receives audio chunks
- Streams or buffers
- Returns partial transcripts

### 5. TranscriptionService (Queueable)

- Final Whisper transcription fallback
- GPT task extraction

---

## 🧠 LLM CONFIGURATION (UPDATED)

PRIMARY MODEL:

- GPT-4.1-mini (task extraction)

FALLBACK:

- GPT-4.1

---

## 📜 TASK EXTRACTION PROMPT (UNCHANGED BUT STRICT)

"You are a system that extracts actionable tasks from transcripts.

Return ONLY valid JSON. No explanations.

If no tasks exist, return [].

Output format:
[
{
"subject": "",
"description": "",
"dueDate": "YYYY-MM-DD or null",
"priority": "High | Normal | Low"
}
]"

---

## 📐 JSON SCHEMA ENFORCEMENT (MANDATORY)

Enforce strict schema validation in Apex:

- Validate JSON structure before Task creation
- Retry once if invalid
- Fallback to GPT-4.1 if needed

---

## 🔁 STREAMING + RETRY LOGIC

1. Stream chunks from LWC
2. Append partial transcript
3. If streaming fails:
   → fallback to Whisper batch
4. If LLM fails:
   → retry GPT-4.1-mini once
   → fallback to GPT-4.1
5. If still fails:
   → mark FAILED

---

## 📦 DATA MODEL (ENHANCED)

Add fields on ContentVersion:

- Transcription_Status\_\_c
- Transcript\_\_c (final)
- Partial_Transcript\_\_c (NEW)
- Streaming_Session_Id\_\_c (NEW)
- Error_Message\_\_c

---

## ⚡ PERFORMANCE + LIMITS

- MUST handle chunked uploads safely
- MUST avoid heap limits
- MUST not exceed callout limits
- MUST debounce streaming calls (important)
- MUST ensure idempotency for chunk processing

---

## 📱 UI/UX REQUIREMENTS

- Must work on Salesforce Mobile + Desktop
- Must be responsive
- Must show real-time feedback
- Must not freeze UI during streaming

---

## 🔐 SECURITY

- Use Named Credentials for OpenAI
- Never expose API keys in LWC
- Sanitize transcript before storing

---

## 📦 OUTPUT REQUIRED

Generate:

1. LWC:
   - voiceNoteRecorder (streaming enabled)
   - voiceIntelligencePanel (new dashboard)

2. Apex:
   - AudioRecorderController
   - StreamingTranscriptionController (NEW)
   - TranscriptionService (Queueable)

3. Flow design (updated for streaming fallback)

4. Named Credential setup

5. Custom Fields

6. Test Classes (75%+ coverage)

7. Deployment instructions

---

## ⚠️ NON-NEGOTIABLE RULES

- MUST be production-ready
- NO pseudo code
- MUST compile in Salesforce
- MUST include error handling
- MUST support reuse across objects
- MUST follow async best practices

---

Now generate the COMPLETE IMPLEMENTATION.
