# StepWise — Project Spec

Shared source of truth for the API contract between the Python backend and the JS frontend. If this changes mid-build, update it here first, then tell your teammate.

## 1. What we're building

An app that turns any tutorial — a PDF, pasted text, or a YouTube link — into a clear, numbered set of steps, then walks the user through it in a chat interface. The user can ask questions about the current step at any time, and when they're ready, they click "Check my work" to have their webcam or screen capture compared against what that step should look like, getting plain-language feedback before moving on.

**Who this is for:** anyone following a tutorial — assembling furniture, following a recipe, learning a piece of software — who wants a clear breakdown of vague or messy instructions, plus a second pair of eyes confirming they're doing it right before they move to the next step, instead of guessing and finding out they made a mistake three steps later.

**Why this is more useful than just asking ChatGPT:** ChatGPT doesn't parse your actual uploaded source into a structured, trackable step list, doesn't know which step you're currently on, and can't look at what you're doing and tell you if it matches. StepWise turns a one-off Q&A into an ongoing, visually-grounded guide.

## 2. Input sources — all normalize to the same pipeline

| Source | How content is extracted | Notes |
|---|---|---|
| PDF | Extract text directly | |
| Typed/pasted text | Used as-is | No extraction needed |
| YouTube URL | Pull existing captions via a transcript API | Only works if the video has captions — test the demo video ahead of time |

All three funnel into plain text, which then goes through the same step-parser LLM call. No real video/audio processing (no ffmpeg, no Whisper) — this is a deliberate scope cut to keep the build reliable in 12 hours.

## 3. Architecture

```
PDF / Text / YouTube URL
            ↓
   [Python] extract plain text (3 small extractor functions, same output shape)
            ↓
   [Python] step parser (LLM call) → numbered step list
            ↓
   [JS] Chat UI — shows steps, current step, lets user ask questions
            ↓
   User clicks "Check my work"
            ↓
   [JS] captures one frame — webcam (physical tasks) or screen (software tasks)
            ↓
   [Python] vision check — image + current step description → pass/fail + feedback
```

## 4. The flow, step by step

1. **Upload** — user submits a PDF, pastes text, or pastes a YouTube URL via `POST /tutorials`
2. **Parsing** — backend extracts plain text, then calls an LLM to produce a numbered list of steps
3. **Walkthrough begins** — frontend shows step 1, user can ask questions about it anytime via `POST /chat/{tutorial_id}`
4. **Check my work** — user clicks the button, frontend captures a frame (webcam or screen, matching the task type), sends it to `POST /check/{tutorial_id}`
5. **Feedback** — backend compares the image against the current step's description using a vision-capable LLM call, returns pass/fail plus a short plain-language note
6. **Advance** — on a pass, frontend moves to the next step; on a fail, the user retries the same step with the feedback shown

## 5. API contract — do not deviate from these names/shapes

| Endpoint | Purpose |
|---|---|
| `POST /tutorials` | Submit a PDF / text / YouTube URL, get back parsed steps |
| `GET /tutorials/{id}` | Get a tutorial's full step list |
| `POST /chat/{tutorial_id}` | Ask a question about the current step |
| `POST /check/{tutorial_id}` | Submit an image to check against the current step |

### Example — submitting a tutorial

Request (PDF or text):
```json
{ "type": "text", "content": "Step one: preheat the oven to 350..." }
```
Request (YouTube):
```json
{ "type": "youtube", "content": "https://youtube.com/watch?v=..." }
```

Response:
```json
{
  "tutorial_id": "xyz789",
  "steps": [
    { "step_id": "s1", "number": 1, "description": "Preheat the oven to 350°F" },
    { "step_id": "s2", "number": 2, "description": "Mix the dry ingredients in a large bowl" }
  ]
}
```

### Example — asking a question

Request:
```json
{ "question": "what if I don't have a large bowl?", "current_step": "s2" }
```
Response:
```json
{ "answer": "Any bowl big enough to hold all the dry ingredients works fine — it doesn't need to be the 'large' size specifically." }
```

### Example — checking work

Request:
```json
{ "step_id": "s2", "image_base64": "<captured frame>" }
```
Response:
```json
{ "pass": false, "feedback": "Looks like the sugar hasn't been added yet — make sure all dry ingredients are in before mixing." }
```

## 6. Who's building what

| Piece | Owner | What it is |
|---|---|---|
| Text/PDF/YouTube extractors | Neal (Python) | Three small functions, same output shape |
| Step parser | Neal (Python) | One LLM call: plain text → numbered steps |
| Chat endpoint | Neal (Python) | Answers questions using tutorial text as context |
| Vision check endpoint | Neal (Python) | Compares an image against the current step |
| Upload UI | Dhiaan (JS) | File picker, text box, URL field — one interface for all three input types |
| Chat interface | Dhiaan (JS) | Shows steps, current step, question box |
| Webcam/screen capture | Dhiaan (JS) | `getUserMedia` (webcam) or `getDisplayMedia` (screen), "Check my work" button |

## 7. Repo structure

```
StepWise/
├── README.md
├── .gitignore
├── docs/
│   └── spec.md
├── backend/                  <- Neal (Python)
│   ├── main.py
│   ├── extractors.py          (pdf_extract, youtube_extract, text_passthrough)
│   ├── step_parser.py
│   ├── chat.py
│   ├── vision_check.py
│   ├── requirements.txt
│   └── .env.example
└── frontend/                   <- Dhiaan + AI (JS)
    ├── index.html
    └── ...
```

## 8. Build order

1. Lock this spec together before writing code
2. Text extractor + step parser — test with plain pasted text first (simplest path)
3. PDF extractor — wire into the same step parser
4. YouTube extractor — wire into the same step parser, test against the actual demo video's captions early
5. Chat endpoint
6. Frontend: upload UI + chat interface, connected to the above
7. Vision check endpoint (Python side)
8. Frontend: webcam/screen capture + "Check my work" button, wired to vision check
9. Full run-through: upload → steps → chat → check → advance, 2-3 times before showing anyone

## 9. Scope cuts — do not build these, we don't have time

- Real video/audio processing (no ffmpeg, no Whisper) — YouTube captions only, no uploaded video files
- Continuous/automatic camera monitoring — manual "Check my work" click only
- More than one vision check per click — one frame, one check, one response
- User accounts / saved tutorial history — each session is fresh
