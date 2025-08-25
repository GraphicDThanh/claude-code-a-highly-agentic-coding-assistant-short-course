### Prompt: give me an overview about this project
```md
This is a Course Materials RAG (Retrieval-Augmented Generation) System - a full-stack web application for querying course materials with AI-powered responses.

Architecture Overview

Backend (Python/FastAPI):
- app.py: Main FastAPI application with API endpoints
- rag_system.py: Core RAG functionality
- vector_store.py: ChromaDB vector database integration
- ai_generator.py: Anthropic Claude AI integration
- document_processor.py: Text processing and chunking
- session_manager.py: User session management
- search_tools.py: Search utilities
- models.py: Data models

Frontend:
- Simple HTML/CSS/JS web interface (frontend/)

Course Data:
- Sample course scripts in docs/ (course1-4_script.txt)

Key Features

1. Semantic Search: Uses ChromaDB for vector storage and similarity search
2. AI Responses: Integrates with Anthropic's Claude API for intelligent answers
3. Session Management: Tracks user conversations
4. Document Processing: Automatically processes course materials
5. REST API: /api/query for questions, /api/courses for statistics
6. Web Interface: Accessible at http://localhost:8000

Tech Stack

- Backend: FastAPI, ChromaDB, Anthropic API, sentence-transformers
- Frontend: Vanilla HTML/CSS/JavaScript
- Package Manager: uv (modern Python package manager)
- Python: 3.13+

The system loads course documents on startup and enables users to ask questions about the course content through a web interface or API.

```

### Prompt: how are these documents processed
```md
Documents are processed through a sophisticated pipeline in document_processor.py:97-259:

Document Processing Pipeline

1. File Reading (read_file:13-21)
- UTF-8 encoding with fallback error handling
- Reads .txt, .pdf, .docx files

2. Metadata Extraction (process_course_document:97-259)
- Line 1: Course Title: [title]
- Line 2: Course Link: [url]
- Line 3: Course Instructor: [instructor]
- Uses regex patterns to parse structured headers

3. Lesson Parsing (lesson_match:166)
- Detects lesson markers: "Lesson X: Title"
- Extracts lesson links if present
- Groups content by lessons

4. Text Chunking (chunk_text:25-91)
- Sentence-based chunking with intelligent splitting
- Configurable chunk size and overlap (from config)
- Smart sentence detection - handles abbreviations
- Overlap calculation - reuses sentences between chunks for context

5. Context Enhancement (chunk_with_context:186,234)
- Adds lesson context: "Course [title] Lesson [X] content: [chunk]"
- First chunk gets: "Lesson X content: [chunk]"
- Preserves course and lesson metadata in each chunk

6. Storage (rag_system.py:38-50)
- Course metadata → vector store for semantic search
- Content chunks → vector database with embeddings
- Deduplication - skips existing courses

The system transforms unstructured course documents into searchable, contextually-rich chunks
ready for semantic retrieval and AI-powered responses.
```

### Prompt: trace the process of handling a user's query from front to backend

```md
Complete Query Processing Flow

1. Frontend Initiation (script.js:45-96)

User types query → sendMessage() triggered
├── Extract query text
├── Disable input/button
├── Add user message to chat
├── Show loading animation
└── POST /api/query with {query, session_id}

2. API Endpoint (app.py:56-74)

FastAPI receives POST /api/query
├── Validate QueryRequest
├── Create/reuse session_id
├── Call rag_system.query(query, session_id)
└── Return QueryResponse{answer, sources, session_id}

3. RAG System Orchestration (rag_system.py:102-140)

query() method processes request
├── Format prompt: "Answer this question about course materials: {query}"
├── Get conversation history from session_manager
├── Call ai_generator.generate_response() with tools
├── Extract sources from tool_manager
├── Update conversation history
└── Return (response, sources)

4. AI Generation (ai_generator.py:43-87)

generate_response() handles Claude interaction
├── Build system prompt + conversation history
├── Prepare API params with tools available
├── Call Claude API with tools enabled
├── If tool_use → execute tools via _handle_tool_execution()
└── Return final response text

5. Tool Execution (ai_generator.py:89-135 + search_tools.py:52-86)

_handle_tool_execution() if Claude wants to search
├── Extract tool calls from Claude response
├── Execute search_course_content tool
│   ├── Call vector_store.search(query, course_name, lesson_number)
│   ├── Format results with course/lesson context
│   └── Store sources in last_sources
├── Send tool results back to Claude
└── Get final response without tools

6. Vector Search (implied from search_tools.py usage)

vector_store.search() performs semantic search
├── Convert query to embeddings
├── Find similar chunks in ChromaDB
├── Filter by course_name/lesson_number if provided
├── Return SearchResults with documents + metadata
└── Handle empty results

7. Frontend Response (script.js:76-95)

Receive API response
├── Update currentSessionId if new
├── Remove loading animation
├── Add assistant message with markdown rendering
├── Show sources in collapsible section
├── Re-enable input/button
└── Scroll to bottom

Key Features:
- Session Management: Maintains conversation context
- Tool-based Search: AI decides when/what to search
- Smart Chunking: Context-aware document processing
- Source Attribution: Tracks which courses/lessons were used
- Error Handling: Graceful failures at each layer
- Real-time UX: Loading states, auto-scroll, markdown rendering

The entire flow is asynchronous and contextual, with the AI making intelligent decisions about
when to search and how to synthesize information.
```

### Prompt: draw a diagram that illustrates this flow
flowchart TD
    %% Frontend Layer
    A[👤 User Types Query] --> B[📱 Frontend JavaScript<br/>script.js:sendMessage()]
    B --> C[🔒 Disable Input<br/>Show Loading Animation]
    C --> D[📡 POST /api/query<br/>{query, session_id}]

    %% API Layer
    D --> E[🚀 FastAPI Endpoint<br/>app.py:/api/query]
    E --> F{Session Exists?}
    F -->|No| G[🆔 Create New Session]
    F -->|Yes| H[📋 RAG System Query]
    G --> H

    %% RAG Orchestration
    H --> I[🧠 RAG System<br/>rag_system.py:query()]
    I --> J[📜 Format Prompt<br/>"Answer this question..."]
    J --> K[🕐 Get Conversation History<br/>session_manager]
    K --> L[🤖 AI Generator<br/>ai_generator.py]

    %% AI Processing
    L --> M[🔧 Build System Prompt<br/>+ Conversation Context]
    M --> N[📞 Call Claude API<br/>with Tools Enabled]
    N --> O{Claude Wants<br/>to Search?}

    %% Tool Execution Branch
    O -->|Yes| P[🔍 Execute search_course_content<br/>search_tools.py]
    P --> Q[📊 Vector Store Search<br/>vector_store.search()]
    Q --> R[🗂️ ChromaDB Similarity Search<br/>+ Course/Lesson Filtering]
    R --> S[📄 Format Results<br/>with Context Headers]
    S --> T[🔄 Send Results to Claude]
    T --> U[💬 Generate Final Response]

    %% Direct Response Branch
    O -->|No| U

    %% Response Processing
    U --> V[📤 Extract Response + Sources<br/>from tool_manager]
    V --> W[💾 Update Session History<br/>session_manager]
    W --> X[📋 Return QueryResponse<br/>{answer, sources, session_id}]

    %% Frontend Response
    X --> Y[📱 Frontend Receives Response<br/>script.js]
    Y --> Z[🔄 Update Session ID<br/>Remove Loading Animation]
    Z --> AA[📝 Add Assistant Message<br/>with Markdown Rendering]
    AA --> BB[📚 Show Sources<br/>in Collapsible Section]
    BB --> CC[✅ Re-enable Input<br/>Auto-scroll Chat]

  ASCII Flow Diagram

  ┌─────────────────────────────────────────────────────────────┐
  │                    FRONTEND LAYER                           │
  ├─────────────────────────────────────────────────────────────┤
  │ User Query → sendMessage() → Show Loading → POST /api/query │
  └─────────────────┬───────────────────────────────────────────┘
                    │
  ┌─────────────────▼───────────────────────────────────────────┐
  │                     API LAYER                               │
  ├─────────────────────────────────────────────────────────────┤
  │ FastAPI Endpoint → Session Check → RAG System Query         │
  └─────────────────┬───────────────────────────────────────────┘
                    │
  ┌─────────────────▼───────────────────────────────────────────┐
  │                 RAG ORCHESTRATION                           │
  ├─────────────────────────────────────────────────────────────┤
  │ Format Prompt → Get History → Call AI Generator             │
  └─────────────────┬───────────────────────────────────────────┘
                    │
  ┌─────────────────▼───────────────────────────────────────────┐
  │                  AI PROCESSING                              │
  ├─────────────────────────────────────────────────────────────┤
  │ Build System Prompt → Claude API → Tool Decision            │
  └─────────────┬───────────────┬───────────────────────────────┘
                │               │
      ┌─────────▼──────────┐   │
      │   TOOL EXECUTION   │   │ Direct Answer
      │                    │   │
      │ Search Tool →      │   │
      │ Vector Store →     │   │
      │ ChromaDB →         │   │
      │ Format Results →   │   │
      │ Back to Claude     │   │
      └─────────┬──────────┘   │
                └───────────────┘
                    │
  ┌─────────────────▼───────────────────────────────────────────┐
  │               RESPONSE PROCESSING                           │
  ├─────────────────────────────────────────────────────────────┤
  │ Extract Sources → Update History → Return Response          │
  └─────────────────┬───────────────────────────────────────────┘
                    │
  ┌─────────────────▼───────────────────────────────────────────┐
  │                FRONTEND RESPONSE                            │
  ├─────────────────────────────────────────────────────────────┤
  │ Update UI → Remove Loading → Show Answer → Display Sources  │
  └─────────────────────────────────────────────────────────────┘

  Key Decision Points

  🤖 Claude's Tool Choice:
  - General knowledge → Direct answer
  - Course-specific → Search first, then answer
  - One search maximum per query

  🔍 Search Intelligence:
  - Semantic similarity matching
  - Course name filtering (partial matches)
  - Lesson number filtering
  - Context-aware result formatting

  📱 UX Flow:
  - Immediate feedback (loading states)
  - Source transparency (collapsible sections)
  - Session continuity (conversation history)
  - Error handling at every layer

### Prompt: how do I run this application?

Quick Start

  1. Set up environment:
  # Create .env file with your Anthropic API key
  echo "ANTHROPIC_API_KEY=your_api_key_here" > .env

  2. Install dependencies:
  uv sync

  3. Run the application:
  chmod +x run.sh
  ./run.sh

  Alternative manual start:
  cd backend
  uv run uvicorn app:app --reload --port 8000

  Access Points

  - Web Interface: http://localhost:8000
  - API Documentation: http://localhost:8000/docs

  Prerequisites

  - Python 3.13+
  - uv package manager
  - Anthropic API key

  The app will automatically load course documents from the docs/ folder on startup.

[Time: 6:25](https://learn.deeplearning.ai/courses/claude-code-a-highly-agentic-coding-assistant/lesson/kkth2/setup-&-codebase-understanding)