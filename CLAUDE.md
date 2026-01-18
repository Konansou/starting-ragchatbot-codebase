# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Retrieval-Augmented Generation (RAG) chatbot system** for querying course materials. Users ask questions through a web UI, and the system retrieves relevant course content using semantic search, then uses Claude AI to synthesize answers with proper source attribution.

**Key Technologies:**
- **Backend**: FastAPI + Python 3.13
- **Vector DB**: ChromaDB (persistent local storage)
- **Embeddings**: Sentence-Transformers (all-MiniLM-L6-v2)
- **AI Model**: Claude (claude-sonnet-4-20250514)
- **Frontend**: Vanilla HTML/CSS/JavaScript

## Development Commands

### Setup
```bash
# Install dependencies
uv sync

# Create .env file with your Anthropic API key
cp .env.example .env
# Edit .env and add: ANTHROPIC_API_KEY=your_key_here
```

### Running the Application
```bash
# Full stack (from project root)
./run.sh

# Or manual backend startup
cd backend
uv run uvicorn app:app --reload --port 8000
```

The application will be available at:
- **Web UI**: http://localhost:8000
- **API Docs**: http://localhost:8000/docs (interactive Swagger UI)

### Testing API Endpoints

**Query a course (POST)**
```bash
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is lesson 1 about?",
    "session_id": null
  }'
```

**Get course statistics (GET)**
```bash
curl http://localhost:8000/api/courses
```

## Architecture Overview

### High-Level Flow

1. **User Query** → Frontend sends to `/api/query`
2. **RAG Pipeline** → Backend coordinates:
   - Session management for conversation context
   - Vector search to find relevant course materials
   - Claude API calls (with tool-based search)
3. **Response Generation** → Claude synthesizes answer from search results
4. **Display** → Frontend shows answer + source citations

### Core Components

**Backend Structure** (`backend/`):
- `app.py` — FastAPI routes (`/api/query`, `/api/courses`)
- `rag_system.py` — Main orchestrator, coordinates all components
- `ai_generator.py` — Claude API interaction with tool calling
- `vector_store.py` — ChromaDB wrapper for semantic search
- `search_tools.py` — Tool definitions for Claude (CourseSearchTool)
- `document_processor.py` — Parses course files and chunks content
- `session_manager.py` — Maintains conversation history per user session
- `models.py` — Pydantic data models (Course, Lesson, CourseChunk)
- `config.py` — Configuration management (API keys, model settings)

**Frontend Structure** (`frontend/`):
- `index.html` — Chat UI with sidebar for course stats
- `script.js` — Query handling, API calls, message rendering
- `style.css` — Styling

### Document Processing Pipeline

Course documents must follow this format:
```
Course Title: Python Basics
Course Link: https://example.com/python
Course Instructor: Jane Smith

Lesson 0: Introduction
Lesson Link: https://example.com/lesson0
[Lesson content...]

Lesson 1: Variables and Data Types
[Lesson content...]
```

**Processing Steps** (document_processor.py):
1. Extract metadata (title, link, instructor) from first 3 lines
2. Split into lessons using "Lesson N:" markers
3. Chunk lesson text into 800-char segments with 100-char overlap
4. Add lesson context to each chunk: `"Lesson N content: {chunk}"`
5. Store as `CourseChunk` objects with metadata (course_title, lesson_number, chunk_index)

### RAG Query Processing

**Two-Turn Claude Interaction:**
1. **Turn 1** — Claude decides if search is needed (tool_choice: auto)
   - System prompt instructs: use search for course-specific questions only
   - Claude generates tool call: `search_course_content(query, course_name, lesson_number)`
2. **Turn 2** — Claude generates final answer
   - Search results formatted with source context
   - Claude synthesizes answer from retrieved chunks
   - Called without tools (prevents infinite loops)

**Tool Search Execution** (search_tools.py):
- CourseSearchTool executes semantic search on `course_content` collection
- Optional filters: course_name, lesson_number
- Returns top 5 results with metadata
- Results formatted as: `[Course Title - Lesson N]\n{chunk_content}`
- Sources extracted and stored for UI display

**Vector Search Details** (vector_store.py):
- Two ChromaDB collections:
  - `course_catalog` — Course metadata for semantic course name matching
  - `course_content` — Actual lesson chunks for content search
- Queries converted to embeddings using Sentence-Transformers
- Similarity computed in embedding space
- Metadata filters applied after semantic search
- Returns distance scores for relevance ranking

### Session Management

- **Session ID**: Generated on first query, persists for multi-turn conversations
- **History Limit**: Last 2 exchanges (configurable via MAX_HISTORY in config.py)
- **Conversation Context**: History formatted as "User: ...\nAssistant: ..." and passed to Claude
- **Memory**: Stored in-memory (SessionManager.sessions dict) — resets on server restart

## Configuration

**Key Settings** (backend/config.py):
- `ANTHROPIC_MODEL` — Claude model version
- `CHUNK_SIZE` — Text chunk size for embeddings (default: 800 chars)
- `CHUNK_OVERLAP` — Context overlap between chunks (default: 100 chars)
- `MAX_RESULTS` — Search results per query (default: 5)
- `MAX_HISTORY` — Conversation exchanges to retain (default: 2)
- `CHROMA_PATH` — Vector DB storage location (default: ./chroma_db)
- `EMBEDDING_MODEL` — Sentence-Transformers model (default: all-MiniLM-L6-v2)

All configurable via environment variables or direct Config class modification.

## Data Flow Details

### Adding Course Materials

**Via API** (rag_system.py:52):
- `RAGSystem.add_course_folder(folder_path)` processes all .pdf, .docx, .txt files
- Deduplicates by course title (skips if already loaded)
- Returns (total_courses, total_chunks) added

**On Startup** (app.py:89):
- Automatically loads documents from `../docs` folder
- Called once when server starts

### Search Behavior

**Metadata Filtering**:
```python
# Course name only
where = {"course_title": "Python Basics"}

# Lesson only
where = {"lesson_number": 2}

# Course + Lesson
where = {"$and": [{"course_title": "..."}, {"lesson_number": ...}]}
```

**Course Name Resolution**:
- If user query mentions course name, semantic search matches against course_catalog
- Returns best matching course title
- Used to filter subsequent content search

## Important Implementation Notes

### Claude API Tool Calling

The system uses Claude's tool calling to handle course search:
- Tool definition includes schema for `query`, optional `course_name`, optional `lesson_number`
- Claude can call tool once per initial query
- Tool results sent back to Claude in second API call
- Final response generated without tools available

### Embedding Model Choice

Uses Sentence-Transformers' `all-MiniLM-L6-v2`:
- Fast, lightweight (33M parameters)
- Good for semantic similarity on general text
- Embeds documents and queries to 384-dim vectors
- No GPU required

### Source Attribution

Sources are extracted from search results metadata and displayed as collapsible section:
- Format: "Course Title - Lesson N"
- Populated during tool execution via CourseSearchTool.last_sources
- Reset after each query to prevent stale sources

## Frontend Specifics

**Session Management (script.js)**:
- Session ID obtained from first `/api/query` response
- Sent with all subsequent queries
- Allows server to maintain conversation history

**Message Rendering**:
- User messages escaped for HTML safety
- Assistant messages parsed as Markdown
- Loading spinner shown during API call
- Messages auto-scroll to bottom

**Course Stats** (script.js:156):
- Loaded on page startup via `/api/courses`
- Displays total course count + course titles
- Updates dynamically if courses added

## Development Patterns

### Adding New Features

**New Tool for Claude**:
1. Create class inheriting from `Tool` (search_tools.py:6)
2. Implement `get_tool_definition()` and `execute(**kwargs)`
3. Register with ToolManager: `tool_manager.register_tool(new_tool)`
4. Claude can now call it in queries

**New Document Type**:
1. Add parsing logic to DocumentProcessor.process_course_document()
2. Return (Course, List[CourseChunk]) tuple
3. Add documents via RAGSystem.add_course_document() or add_course_folder()

**New Config Setting**:
1. Add to Config dataclass in config.py
2. Can read from environment or use default
3. Access via `config.SETTING_NAME` throughout application

### Debugging

**ChromaDB Issues**:
- Vector DB stored at `./chroma_db` (local directory)
- Delete entire directory to reset: `rm -rf chroma_db`
- Server will recreate on next startup

**Session Issues**:
- Sessions stored in memory only (SessionManager.sessions dict)
- Cleared on server restart
- For testing: manually create sessions or pass session_id=null

**Claude API Issues**:
- Check ANTHROPIC_API_KEY environment variable
- Verify API key is valid at https://console.anthropic.com
- Check token limits (max 800 tokens per response in current config)

## Common Tasks

**Load new course materials:**
```python
# Option 1: Via running server
curl -X POST http://localhost:8000/api/documents \
  -F "file=@course.pdf"

# Option 2: Manually place in docs/ folder, server loads on startup
```

**Query with course filter:**
```bash
curl -X POST http://localhost:8000/api/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is RAG?",
    "session_id": "session_1"
  }'
# Claude will recognize course reference in query and use search_course_content tool
```

**Check API documentation:**
- Navigate to http://localhost:8000/docs while server running
- Interactive Swagger UI shows all endpoints and request/response schemas

## Known Limitations

- **Session Memory**: Conversations reset on server restart (in-memory storage only)
- **Embedding Quality**: all-MiniLM-L6-v2 is good but not state-of-the-art (trade-off for speed)
- **Search Depth**: Returns top 5 results only (configurable, but higher values increase latency)
- **Document Format**: Strict parsing of lesson markers (must use "Lesson N:" format)
- **Multi-language**: Sentence-Transformers trained primarily on English
