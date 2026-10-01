# StepWise

Turn any tutorial — a PDF, pasted text, or a YouTube link — into clear, numbered steps. Ask questions as you go, and click "Check my work" to have your webcam or screen compared against what the current step should look like.

Built for [event name] by Neal and Dhiaan.

## What it does

1. Upload a PDF, paste in text, or paste a YouTube link
2. StepWise parses it into a numbered list of steps
3. Chat with it about the current step anytime
4. Click "Check my work" — it captures your webcam (physical tasks) or screen (software tasks) and tells you if you're on track
5. Pass → move to the next step. Fail → get specific feedback and retry

Full architecture and API contract: [`docs/spec.md`](docs/spec.md)

## Project structure

```
StepWise/
├── backend/        Python (FastAPI) — parsing, chat, vision check
├── frontend/        JS — upload UI, chat, camera/screen capture
└── docs/spec.md       Full spec: architecture, API contract, build order
```

## Running it locally

### Backend
```bash
cd backend
pip install -r requirements.txt --break-system-packages
uvicorn main:app --reload
```
Runs at `http://localhost:8000`. Interactive API docs at `http://localhost:8000/docs`.

### Frontend
```bash
cd frontend
# open index.html directly, or serve it:
python3 -m http.server 3000
```
Then open `http://localhost:3000`.

### Environment variables
Copy `backend/.env.example` to `backend/.env` and fill in your actual API key(s) — never commit the real `.env` file.

## API quick reference

| Endpoint | Purpose |
|---|---|
| `POST /tutorials` | Submit a PDF / text / YouTube URL, get back parsed steps |
| `GET /tutorials/{id}` | Get a tutorial's full step list |
| `POST /chat/{tutorial_id}` | Ask a question about the current step |
| `POST /check/{tutorial_id}` | Submit an image to check against the current step |

Full request/response examples in [`docs/spec.md`](docs/spec.md).

## Team

| Piece | Owner |
|---|---|
| Backend (extractors, step parser, chat, vision check) | Neal |
| Frontend (upload UI, chat interface, camera/screen capture) | Dhiaan |

## Status
- [ ] Text input → step parser
- [ ] PDF extraction
- [ ] YouTube transcript extraction
- [ ] Chat endpoint
- [ ] Upload UI
- [ ] Chat interface
- [ ] Vision check endpoint
- [ ] Webcam/screen capture + Check my work
- [ ] Full end-to-end run-through
