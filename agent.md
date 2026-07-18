# AskDocAI Project Agent Notes

## Purpose

This repository contains a PDF question-answering application. Users upload a PDF, the backend extracts and chunks the text, a retrieval layer finds relevant context, and a language model generates answers that can be returned in English, Hindi, or Marathi.

## Current Naming

- Repository/product name in the root README: `AskDocAI`
- Backend app title in `backend/main.py`: `BhashaSetu - Multilingual Smart Document Assistant`
- The codebase currently mixes both names, so any future cleanup should keep that inconsistency in mind.

## High-Level Architecture

1. The React frontend renders a landing page and a chat/upload flow.
2. The frontend sends file upload and chat requests to the FastAPI backend over `/api`.
3. The backend validates and stores the uploaded PDF temporarily, extracts text, chunks it, and stores the chunks in an in-memory RAG pipeline per session.
4. When a question arrives, the backend retrieves the most relevant chunks for that session and asks the LLM to generate a response.
5. Answers and session chat history are stored in memory for the lifetime of the backend process.

## Repository Structure

### Root

- `README.md` concise project entry point
- `agent.md` detailed internal project notes
- `QUICKSTART.bat` Windows helper script
- `QUICKSTART.sh` shell helper script

### Backend

- `backend/main.py` FastAPI application setup, middleware, router registration, startup/shutdown hooks
- `backend/run_server.py` backend runner script
- `backend/requirements.txt` Python dependencies
- `backend/api/routes.py` API endpoints
- `backend/api/schemas.py` request and response schemas
- `backend/api/utils.py` upload and file utility helpers
- `backend/models/session_store.py` in-memory session storage
- `backend/services/pdf_processor.py` PDF extraction, cleaning, and chunking
- `backend/services/rag_pipeline.py` retrieval pipeline and vector collection management
- `backend/services/gemini_service.py` LLM response generation
- `backend/services/deepseek_service.py` alternate LLM service integration
- `backend/services/embedding_service.py` embedding generation
- `backend/services/language_detector.py` language detection helpers
- `backend/services/translator.py` translation support
- `backend/services/mock_responses.py` test or fallback responses
- `backend/services/__init__.py` package marker
- `backend/api/__init__.py` package marker
- `backend/models/__init__.py` package marker

### Frontend

- `frontend/index.html` Vite entry HTML
- `frontend/package.json` frontend scripts and dependencies
- `frontend/vite.config.js` Vite config
- `frontend/tailwind.config.js` Tailwind config
- `frontend/postcss.config.js` PostCSS config
- `frontend/src/main.jsx` React bootstrap
- `frontend/src/App.jsx` top-level page switch and providers
- `frontend/src/index.css` global styles
- `frontend/src/context/ThemeContext.jsx` theme state
- `frontend/src/context/LanguageContext.jsx` language state
- `frontend/src/pages/Home.jsx` marketing/landing page
- `frontend/src/pages/ChatPage.jsx` upload/chat flow container
- `frontend/src/components/Navbar.jsx` top navigation
- `frontend/src/components/Upload.jsx` PDF upload UI
- `frontend/src/components/Chat.jsx` chat UI
- `frontend/src/components/LanguageSelector.jsx` language selection UI
- `frontend/src/components/Footer.jsx` footer
- `frontend/src/services/api.js` frontend API client
- `frontend/src/assets/PDFAnalysis.jsx` illustration used on the landing page

## Backend Details

### Entry Point

`backend/main.py` creates the FastAPI app, configures CORS for `http://localhost:5173` and `http://localhost:3000`, optionally enables gzip compression, loads environment variables, and includes the API router under `/api`.

### Optional Voice Module

`backend/main.py` attempts to import `speech_module.routes`. If that import fails, the app continues without voice routes. This module is optional and not required for the PDF chat flow.

### Session Model

The backend keeps two in-memory structures that matter for behavior:

- `session_store` tracks session metadata and chat history.
- `rag_pipelines` maps a session ID to its active retrieval pipeline.

Because both live in memory, sessions are not persistent across process restarts.

### API Endpoints

Defined in `backend/api/routes.py`:

- `GET /api/health` checks whether the Gemini service can initialize.
- `POST /api/upload-pdf` validates a PDF, extracts text, chunks it, creates a session, builds a RAG pipeline, and stores session metadata.
- `POST /api/ask-question` retrieves context for a session, sends the question plus context to Gemini, and stores the Q&A pair in session history.
- `POST /api/translate` translates text into `en`, `hi`, or `mr`.
- `GET /api/session/{session_id}` returns session metadata and chat history.
- `DELETE /api/session/{session_id}` removes the session and its pipeline.

### Backend Flow

1. Validate upload as a PDF.
2. Save the file temporarily.
3. Extract raw text from the PDF.
4. Clean and chunk the text.
5. Create a new RAG collection for the session.
6. Add the chunks to the pipeline.
7. Store session metadata and document text.
8. On question requests, fetch context with `top_k=3`.
9. Send prompt, context, and requested language to Gemini.
10. Store user and assistant messages in the session history.

### Validation Rules

- Uploads must be PDF files.
- Questions only accept `en`, `hi`, or `mr`.
- Translation only accepts `en`, `hi`, or `mr`.
- The frontend enforces a 50 MB upload limit.

### Runtime Behavior

- Temporary upload files are cleaned up after processing.
- On upload failure, the session and pipeline are deleted.
- The health endpoint returns unhealthy if the Gemini service cannot initialize.
- The backend startup log checks whether `DEEPSEEK_API_KEY` is present, even though the active route layer currently uses Gemini for answers.

## Frontend Details

### App Flow

`frontend/src/App.jsx` uses context providers for theme and language, then switches between the home page and the chat page with local state.

### Home Page

`frontend/src/pages/Home.jsx` contains the marketing landing page with a hero section, feature grid, how-it-works section, audience section, and call to action.

### Chat Page

`frontend/src/pages/ChatPage.jsx` manages the document upload state and then renders the chat interface once a session is created.

### Upload and Chat

- `frontend/src/components/Upload.jsx` handles file selection, PDF validation, size validation, and upload requests.
- `frontend/src/components/Chat.jsx` is the main conversation surface after upload.
- `frontend/src/components/LanguageSelector.jsx` updates the current language.

### API Client

`frontend/src/services/api.js` sends requests to `/api` and provides helpers for:

- health checks
- PDF uploads
- question answering
- translation
- session retrieval
- session deletion

## Dependencies and Tooling

### Frontend

- React 18
- React Router DOM
- Axios
- Vite
- Tailwind CSS
- PostCSS and Autoprefixer

### Backend

- FastAPI
- Uvicorn
- python-dotenv
- PDF extraction and RAG-related libraries declared in `backend/requirements.txt`

## Environment Expectations

Likely required or expected environment values include:

- `DEEPSEEK_API_KEY`
- Any Gemini-related key or configuration used by `backend/services/gemini_service.py`
- Optional settings consumed by `backend/services/translator.py` or other service modules

`backend/main.py` loads environment variables from `.env` at startup.

## Local Development

### Backend

Run from `backend/` with a virtual environment and `uvicorn main:app --reload --host 0.0.0.0 --port 8000`.

### Frontend

Run from `frontend/` with `npm install` and `npm run dev`.

### Browser URL

- Frontend: `http://localhost:5173`
- Backend: `http://localhost:8000`
- API docs: `http://localhost:8000/docs`

## Behavior Constraints

- The system is session-based and in-memory.
- It does not persist document indexes across restarts.
- It currently supports PDF only.
- Session cleanup matters because stale sessions are otherwise kept only in memory.

## Good Places to Edit

- UI behavior: `frontend/src/App.jsx`, `frontend/src/pages/ChatPage.jsx`, `frontend/src/components/Upload.jsx`, `frontend/src/components/Chat.jsx`
- API behavior: `backend/api/routes.py`
- PDF/RAG logic: `backend/services/pdf_processor.py`, `backend/services/rag_pipeline.py`
- Session logic: `backend/models/session_store.py`
- Service integration: `backend/services/gemini_service.py`, `backend/services/deepseek_service.py`, `backend/services/translator.py`

## Notes For Future Cleanup

- Decide whether the project name should be standardized as AskDocAI or BhashaSetu.
- Consider documenting the expected backend model provider more explicitly because the current code mentions DeepSeek in some places and Gemini in others.
- If persistence is needed later, move session and vector state out of memory.