# CodeMentor AI 🚀

**AWS AI Hackathon 2026** — An AI-powered code mentor that helps CS students prepare for placements through RAG-driven code review, personalized roadmaps, and adaptive coding games.

---

## 🧠 AI Architecture

CodeMentor AI uses a **RAG (Retrieval-Augmented Generation)** pipeline powered by **Google Gemini** and **FAISS**:

```
React Frontend
     ↓
FastAPI Backend
     ↓
RAG Retrieval (FAISS vector search → Knowledge Base in PostgreSQL)
     ↓
LLM Processing (Google Gemini 1.5 Flash)
     ↓
Structured JSON Response
     ↓
Stored in PostgreSQL
     ↓
Displayed in Frontend UI
```

### How LLM Integrates with RAG

| Feature | RAG Used | LLM Used | Cached |
|---------|----------|----------|--------|
| Code Review | ✅ Top-K FAISS chunks | ✅ Gemini 1.5 Flash | ❌ (real-time) |
| Game Questions | ✅ Concept-matched KB chunks | ✅ Gemini 1.5 Flash | ✅ DB cache (6h TTL) |
| Roadmap Generation | ✅ Weak-concept KB chunks | ✅ Gemini 1.5 Flash | ✅ DB versioned |
| Syllabus Q&A | ✅ User documents (FAISS) | ✅ Gemini 1.5 Flash | ❌ |
| Analytics | ❌ | ❌ | Rule-based |
| Auth / Leaderboard | ❌ | ❌ | DB-driven |

### LLM Flow — Code Review

```
User submits code
     ↓
Pattern Service: detect weakest concept
     ↓
RAG: retrieve top-3 FAISS chunks relevant to concept
     ↓
Prompt = context + code + student profile
     ↓
Gemini 1.5 Flash → structured JSON feedback
     ↓
Validate JSON → retry once on invalid
     ↓
Save to submissions table + update error_patterns
     ↓
Return feedback JSON to frontend
```

### LLM Flow — Game Questions (with Caching)

```
GET /api/game/question/{game_type}?user_id=...
     ↓
Identify weakest concept from error_patterns
     ↓
Check game_questions cache (6h TTL) → return if fresh hit
     ↓
RAG: fetch knowledge chunks for that concept
     ↓
Gemini: generate question with RAG context
     ↓
Cache in game_questions table → return to frontend
```

### LLM Flow — Roadmap Generation

```
GET /api/roadmap/{user_id}?regenerate=true
     ↓
Top 3 weakest concepts from error_patterns
     ↓
RAG: fetch KB chunks matching each concept
     ↓
Gemini: generate 6-week roadmap using user profile + KB resources
     ↓
Save versioned roadmap to roadmaps table → return
```

---

## 🛠 Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React + Vite + Material UI |
| **Backend** | FastAPI (Python 3.11) |
| **Database** | PostgreSQL |
| **Vector Store** | FAISS (local, auto-synced with DB) |
| **Embeddings** | Google Gemini `models/embedding-001` |
| **LLM** | Google Gemini `gemini-1.5-flash` |
| **Auth** | JWT (demo tokens) |

---

## ⚙️ Environment Variables

Create a `.env` file in `backend/`:

```env
# ── AI (required) ─────────────────────────────────────────────
GEMINI_API_KEY=your_google_gemini_api_key_here
MODEL_NAME=gemini-1.5-flash
EMBEDDING_MODEL=models/embedding-001

# ── Database ──────────────────────────────────────────────────
DATABASE_URL=postgresql://postgres:yourpassword@localhost:5432/codementor_ai

# ── FAISS ─────────────────────────────────────────────────────
FAISS_INDEX_PATH=./knowledge_base/faiss_index
USER_FAISS_INDEX_PATH=./knowledge_base/user_faiss_index

# ── Security ──────────────────────────────────────────────────
SECRET_KEY=your_32_char_random_secret_key_here
```

**Get your free Gemini API key:** [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey)

---

## 💻 Installation

### 1. Database Setup

```sql
-- In pgAdmin or psql:
CREATE DATABASE codementor_ai;
```

> SQLAlchemy auto-creates all tables (`users`, `submissions`, `error_patterns`, `knowledge_base`, `roadmaps`, `game_questions`, etc.) on first startup.

### 2. Backend Setup (Python 3.11)

```powershell
cd backend

# Create Python 3.11 virtual environment
py -3.11 -m venv .venv

# Activate
.venv\Scripts\activate      # Windows
# source .venv/bin/activate # macOS/Linux

# Install dependencies
pip install -r requirements.txt

# Run the server
.venv\Scripts\python.exe -m uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

> ⚠️ Always run **one single uvicorn instance**. Multiple processes on port 8000 will cause CORS hangs.

Backend runs at: **http://localhost:8000**  
API docs (Swagger): **http://localhost:8000/docs**

### 3. Build the Knowledge Base Index

```powershell
cd backend
.venv\Scripts\python.exe knowledge_base/build_kb.py
```

This embeds 88 curated programming knowledge chunks and saves a FAISS index to `./knowledge_base/faiss_index`.

### 4. Frontend Setup

```powershell
# From project root
npm install
npm run dev
```

Frontend runs at: **http://localhost:5173**

---

## 🔌 Key API Endpoints

| Method | Endpoint | Description |
|--------|---------|-------------|
| `POST` | `/api/auth/register` | Register new user |
| `POST` | `/api/auth/login` | Login → JWT token |
| `POST` | `/api/code/submit` | Submit code for AI review |
| `GET` | `/api/roadmap/{user_id}` | Get/regenerate roadmap |
| `GET` | `/api/game/question/{type}` | Get adaptive game question |
| `POST` | `/api/ask` | Ask question from syllabus |
| `POST` | `/api/documents/upload` | Upload notes/syllabus |
| `GET` | `/api/analytics/{user_id}` | Get dashboard metrics |
| `GET` | `/api/documents/rag-status` | FAISS index health |

---

## 🗂 Project Structure

```
code-mentor-ai/
├── backend/
│   ├── app/
│   │   ├── models/           # SQLAlchemy models
│   │   │   ├── user.py
│   │   │   ├── submission.py
│   │   │   ├── error_pattern.py
│   │   │   ├── roadmap.py
│   │   │   ├── knowledge_base.py
│   │   │   └── game_question.py  # ← question cache
│   │   ├── routes/           # FastAPI routers
│   │   │   ├── auth.py
│   │   │   ├── code.py
│   │   │   ├── game.py
│   │   │   ├── roadmap.py
│   │   │   ├── documents.py
│   │   │   └── ask.py
│   │   └── services/         # Business logic + AI
│   │       ├── llm_service.py      # Code review (Gemini + RAG)
│   │       ├── game_service.py     # Game Qs (Gemini + RAG + cache)
│   │       ├── roadmap_service.py  # Roadmaps (Gemini + RAG)
│   │       ├── rag_service.py      # Document Q&A
│   │       └── pattern_service.py  # Error DNA tracking
│   ├── rag.py               # FAISS index management
│   ├── knowledge_base/
│   │   └── build_kb.py      # KB indexing script
│   ├── requirements.txt
│   └── .env                 # ← your secrets here
└── src/                     # React frontend
    └── pages/
        ├── CodeSubmit.jsx
        ├── Games.jsx
        ├── Roadmap.jsx
        ├── AskSyllabus.jsx
        └── ...
```

---

## 🛡 Error Handling

All LLM calls include:
- **Retry logic** (2 attempts for code review)
- **Safe JSON validation** before returning to frontend
- **Static fallback** responses when Gemini is unavailable
- **Graceful logging** of all errors (`logger.error(...)`)

No API error will crash the backend — all return a structured `{"error": "..."}` response.
