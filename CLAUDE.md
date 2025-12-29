# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a RAG (Retrieval-Augmented Generation) chatbot system for querying course materials. It uses Claude's tool calling capabilities to perform semantic search over vectorized course content stored in ChromaDB.

**Tech Stack**: FastAPI backend, vanilla JavaScript frontend, Claude Sonnet 4 with tool use, ChromaDB with SentenceTransformers embeddings.

## Development Commands

### Setup
```bash
# Install uv package manager (if needed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Install dependencies
uv sync

# Configure environment
cp .env.example .env
# Edit .env and add your ANTHROPIC_API_KEY
```

### Running the Application
```bash
# Option 1: Using the run script
./run.sh

# Option 2: Manual start with auto-reload
cd backend && uv run uvicorn app:app --reload --port 8000
```

**Access Points**:
- Web Interface: `http://localhost:8000`
- API Documentation: `http://localhost:8000/docs` (auto-generated Swagger UI)

### API Endpoints
- `POST /api/query` - Submit queries with optional session_id
- `GET /api/courses` - Get course analytics

## Architecture

### Core Components (3-tier architecture)

**1. API Layer** (`backend/app.py`)
- FastAPI application serving REST endpoints and static frontend
- Loads course documents from `../docs` folder on startup
- Serves at `http://localhost:8000`

**2. RAG Orchestrator** (`backend/rag_system.py`)
- Central coordinator integrating all components
- Key methods:
  - `add_course_folder()` - Process documents with deduplication
  - `query()` - Main query processing with tool-based search
  - `get_course_analytics()` - Return statistics

**3. AI Generator** (`backend/ai_generator.py`)
- Handles Claude API interactions with tool calling workflow
- Two-phase process:
  1. Initial call with tool definitions
  2. If `stop_reason == "tool_use"`, execute tools and make follow-up call with results
- Configuration: `claude-sonnet-4-20250514`, temperature=0, max_tokens=800

**4. Tool System** (`backend/search_tools.py`)
- Abstract `Tool` base class for extensibility
- `CourseSearchTool` - Semantic search implementation
- `ToolManager` - Registry pattern for tool execution and source tracking

**5. Vector Store** (`backend/vector_store.py`)
- Two ChromaDB collections:
  - `course_catalog` - Course metadata for fuzzy name resolution
  - `course_content` - Chunked course material with embeddings
- Performs fuzzy course name matching (e.g., "MCP" finds full course title)
- Builds filters for course/lesson constraints

**6. Document Processor** (`backend/document_processor.py`)
- Parses structured course documents with format:
  ```
  Course Title: [title]
  Course Link: [url]
  Course Instructor: [name]
  Lesson N: [title]
  Lesson Link: [url]
  [content...]
  ```
- Sentence-based chunking: 800 chars with 100 char overlap
- Adds contextual prefixes to chunks for better retrieval

**7. Session Manager** (`backend/session_manager.py`)
- In-memory conversation history (not persisted)
- Default: stores last 2 exchanges per session
- Session lifecycle management

**8. Data Models** (`backend/models.py`)
- Pydantic models: `Course`, `Lesson`, `CourseChunk`

**9. Frontend** (`frontend/`)
- `index.html` - Sidebar with course stats and suggested questions
- `script.js` - API calls, markdown rendering, session management
- `style.css` - Modern dark theme

### Query Flow

1. User submits query → Frontend (`script.js`)
2. POST to `/api/query` with `{query, session_id}`
3. RAGSystem retrieves conversation history from SessionManager
4. AIGenerator makes initial Claude API call with tool definitions
5. **If Claude uses `search_course_content` tool**:
   - ToolManager executes CourseSearchTool
   - VectorStore performs fuzzy course name matching
   - ChromaDB semantic search with filters
   - Results formatted with source citations
6. AIGenerator makes follow-up Claude call with tool results
7. SessionManager saves exchange
8. Response returned with answer + sources
9. Frontend displays markdown with collapsible sources

See `query-flow-diagram.md` for detailed sequence and data flow diagrams.

### Key Architecture Decisions

- **Tool-Based Search**: Uses Claude's native tool calling instead of direct context injection
- **Dual Collections**: Separates course metadata from content for efficient fuzzy matching
- **Stateless Sessions**: In-memory storage, resets on server restart
- **Smart Chunking**: Sentence-based with overlap preserves context across boundaries
- **Lightweight Embeddings**: `all-MiniLM-L6-v2` balances speed and quality

## Configuration

All configuration in `backend/config.py`:
- AI Model: `claude-sonnet-4-20250514`
- Embedding Model: `all-MiniLM-L6-v2`
- Chunk Size: 800 characters
- Chunk Overlap: 100 characters
- Max Search Results: 5
- Max Conversation History: 2 messages
- ChromaDB Path: `./chroma_db`

## Adding Course Materials

Place `.txt` files in `docs/` folder with the structured format expected by DocumentProcessor. The application auto-loads all documents from `docs/` on startup.

## Important Notes

- Python 3.13+ required (specified in `.python-version`)
- Uses `uv` for fast, modern Python package management
- ChromaDB creates persistent storage in `backend/chroma_db/`
- Sessions are in-memory only and not persisted across restarts
- No testing, linting, or CI/CD infrastructure currently configured
