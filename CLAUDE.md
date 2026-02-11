# RAG Chatbot Query Flow: Frontend to Backend

## Overview

This document traces the complete journey of a user's query from the frontend interface through the backend RAG system to vector storage and AI generation, then back to the user with an intelligent response and sources.

---

## 1. FRONTEND - User Input (script.js)

```
User types question → clicks Send button
                  ↓
chatInput.value captured → sendMessage() function triggered
```

**Key code** (script.js:45-72):
- User enters query in `chatInput` element
- `sendMessage()` function extracts the query
- Loading message displayed while waiting
- POST request to `/api/query` with:
  ```json
  {
    "query": "user's question",
    "session_id": "current_session_id_or_null"
  }
  ```

**Flow**:
1. User types in input field
2. Presses Enter or clicks Send button
3. Event listener triggers `sendMessage()`
4. Input disabled (prevent duplicate submissions)
5. User message added to chat display
6. Loading spinner shown
7. Async fetch request sent to backend

---

## 2. BACKEND - FastAPI Router (app.py)

```
HTTP POST /api/query received
          ↓
QueryRequest validation (Pydantic model)
          ↓
rag_system.query() called
```

**Key code** (app.py:56-72):
- FastAPI validates request with `QueryRequest` model
- Creates new session if `session_id` is None
- Calls `rag_system.query(query, session_id)`

**Endpoint details**:
- Route: `POST /api/query`
- Middleware: CORS enabled, TrustedHost configured
- Response model: `QueryResponse` with answer, sources, session_id
- Error handling: HTTPException with 500 status on failure

---

## 3. RAG System Orchestration (rag_system.py)

```
rag_system.query() receives query & session_id
           ↓
Gets conversation history from SessionManager (if session exists)
           ↓
Calls ai_generator.generate_response() with:
  - Query
  - Conversation history
  - Tool definitions (search_course_content tool)
  - Tool manager instance
```

**Key code** (rag_system.py:102-140):
- Retrieves previous conversation context if session exists
- Passes tools to Claude AI for intelligent decision-making
- Collects sources from tool execution
- Updates session with new exchange

**Flow**:
1. Check if session_id exists
2. If yes, retrieve conversation history
3. Prepare prompt: "Answer this question about course materials: {query}"
4. Call AIGenerator with tools enabled
5. Extract sources from tool manager
6. Save exchange to session history
7. Return response + sources tuple

---

## 4. AI Generation with Tool Use (ai_generator.py)

```
ai_generator.generate_response() called
           ↓
Builds system prompt + conversation history
           ↓
Calls Claude API with:
  - System prompt (RAG instructions)
  - User query
  - Available tools (search_course_content)
  - temperature: 0 (deterministic)
  - max_tokens: 800
```

**System Prompt Instructions**:
- Use search tool ONLY for course-specific questions
- One search per query maximum
- No meta-commentary about search process
- Direct answers to user questions
- Keep responses brief and focused

**Decision Point**:
- Claude analyzes if search is needed
- If `stop_reason == "tool_use"`: Execute tool
- If direct answer: Return response immediately

**Tool Calling Flow** (ai_generator.py:83-135):
1. Claude decides whether to use `search_course_content` tool
2. If `stop_reason == "tool_use"`:
   - Extract tool call parameters
   - Execute tool via ToolManager
   - Collect tool results
   - Make follow-up API call with tool results
   - Return final generated response
3. Else:
   - Return direct response text

---

## 5. Tool Execution (search_tools.py)

If Claude decides to search:

```
Claude selects search_course_content tool
              ↓
Tool receives parameters:
  - query: "what to search for"
  - course_name: (optional, e.g., "MCP", "Introduction")
  - lesson_number: (optional, e.g., 1, 2, 3)
              ↓
ToolManager.execute_tool() called
              ↓
CourseSearchTool.execute() invoked
```

**Tool Definition** (search_tools.py:27-50):
```json
{
  "name": "search_course_content",
  "description": "Search course materials with smart course name matching and lesson filtering",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "What to search for" },
      "course_name": { "type": "string", "description": "Course title (partial matches work)" },
      "lesson_number": { "type": "integer", "description": "Specific lesson number" }
    },
    "required": ["query"]
  }
}
```

**Execution Flow** (search_tools.py:52-114):
1. Validate parameters
2. Call `vector_store.search()` with filters
3. Format results with course/lesson context
4. Track source information for UI display
5. Return formatted search results as string

**Result Formatting**:
- Each result includes: [Course Title - Lesson X] + content
- Sources tracked: course title + lesson number
- Empty results handled: explicit "No relevant content found" message

---

## 6. Vector Search (vector_store.py)

```
VectorStore.search() called
         ↓
Step 1: Resolve course name (if provided)
        - Semantic search in course_catalog collection
        - Matches user-provided course name to actual title
         ↓
Step 2: Build ChromaDB filter
        - Filter by course_title and/or lesson_number
         ↓
Step 3: Execute semantic search in course_content collection
        - Query embedded using SentenceTransformer
        - Top 5 results returned (configurable max_results)
         ↓
Returns SearchResults object:
  - documents: [content chunks]
  - metadata: [course_title, lesson_number, chunk_index]
  - distances: [relevance scores]
```

**Key code** (vector_store.py:61-133):
- `search()`: Main interface, orchestrates resolution and search
- `_resolve_course_name()`: Semantic search to find course title
- `_build_filter()`: Constructs ChromaDB where clause
- `SearchResults`: Dataclass wrapping ChromaDB results

**Collections**:
1. **course_catalog**: Stores course metadata
   - IDs: course titles
   - Documents: course title (for semantic search)
   - Metadata: instructor, course_link, lessons_json, lesson_count

2. **course_content**: Stores document chunks
   - IDs: "{course_title}_{chunk_index}"
   - Documents: content text
   - Metadata: course_title, lesson_number, chunk_index

**Search Parameters**:
- `query`: Main search text (required)
- `course_name`: Optional filter (partial matching)
- `lesson_number`: Optional filter (exact matching)
- `limit`: Override default max_results (default: 5)

**Filtering Logic**:
- No filters: Search all content
- Course only: `{"course_title": course_title}`
- Lesson only: `{"lesson_number": lesson_number}`
- Both: `{"$and": [{"course_title": ...}, {"lesson_number": ...}]}`

---

## 7. Embedding & ChromaDB Lookup

**Embedding Model**: SentenceTransformer (default: `all-MiniLM-L6-v2`)
- Converts text queries to 384-dimensional vectors
- Used for semantic similarity matching
- Same model for both encoding and searching

**ChromaDB Query** (vector_store.py:93-97):
```python
results = self.course_content.query(
    query_texts=[query],
    n_results=search_limit,
    where=filter_dict
)
```

**Result Handling**:
- Results converted to `SearchResults` object
- Empty results: `SearchResults.empty()` with error message
- Error handling: Try-except returns error message

---

## 8. AI Response Generation (ai_generator.py)

```
Claude receives tool results
         ↓
Processes search results in context
         ↓
Generates final response following system prompt:
  - Brief and focused
  - Educational value
  - No meta-commentary about search
  - Direct answer to user question
         ↓
Returns text response
```

**Response Characteristics**:
- Temperature: 0 (deterministic, no randomness)
- Max tokens: 800 (concise responses)
- System prompt guides output format
- Follow-up call after tool execution synthesizes results

---

## 9. Session Management (session_manager.py)

```
After response generated:
      ↓
Add exchange to conversation history:
  - Query added
  - Response added
      ↓
Stored in session for context in future queries
```

**Session Persistence**:
- SessionManager maintains conversation history
- Each session has unique ID
- History limited by MAX_HISTORY config
- Used for context in subsequent queries
- Enables multi-turn conversations

---

## 10. BACKEND Response (app.py)

```
QueryResponse built and returned:
{
  "answer": "AI's response text",
  "sources": ["Course Title - Lesson X", ...],
  "session_id": "session_id_for_next_request"
}
```

**Response Model** (app.py:43-47):
- `answer`: Generated AI response text
- `sources`: List of source references (course + lesson)
- `session_id`: For maintaining conversation continuity

**HTTP Response**:
- Status: 200 OK (success) or 500 Internal Server Error (failure)
- Content-Type: application/json
- CORS headers enabled

---

## 11. FRONTEND - Display Results (script.js)

```
HTTP response received
        ↓
Replace loading message with AI response
        ↓
Display sources in collapsible panel
        ↓
Update session_id for next query
        ↓
Re-enable input for next message
```

**Key code** (script.js:63-95):
- Check response status
- Parse JSON response
- Update currentSessionId if null
- Remove loading spinner
- Add assistant message with response
- Attach sources if available
- Re-enable input fields
- Set focus on input

**Message Display**:
- User messages: HTML-escaped, plain text
- Assistant messages: Markdown parsed (using marked.js)
- Sources: Collapsible details element with summary "Sources"

---

## Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ FRONTEND (Browser - JavaScript)                                 │
│ User Query → sendMessage() → POST /api/query                   │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ BACKEND (FastAPI - app.py)                                      │
│ Route: POST /api/query → RAGSystem.query()                     │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ RAG ORCHESTRATOR (rag_system.py)                                │
│ - Get session history                                           │
│ - Prepare tools for Claude                                      │
│ - Call AI Generator with tool definitions                       │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ AI GENERATOR (ai_generator.py)                                  │
│ - Call Claude API with tools                                    │
│ - Decision: Search needed? YES/NO                               │
└──────────────────┬──────────────────────────────────────────────┘
                   │
          ┌────────┴────────┐
          │                 │
        YES               NO
          │                 │
          ↓                 ↓
    ┌──────────┐      ┌─────────┐
    │Tool Use  │      │Direct   │
    │          │      │Answer   │
    └────┬─────┘      └────┬────┘
         │                 │
         ↓                 │
    ┌──────────────────────┐
    │ Execute Tool (search_tools.py)     │
    │ → Vector Search                    │
    └──────────┬───────────┘
               │
               ↓
    ┌─────────────────────────────────┐
    │ VectorStore (vector_store.py)   │
    │ - Resolve course names          │
    │ - Build ChromaDB filters        │
    │ - Semantic search               │
    │ - Return top-k results          │
    └──────────┬────────────────────┘
               │
               ↓
    ┌─────────────────────────────────┐
    │ ChromaDB Collections            │
    │ - course_catalog (metadata)     │
    │ - course_content (chunks)       │
    └─────────────────────────────────┘
               │
               └──────────────┬─────────────┐
                              │             │
                    ┌─────────▼──────┐      │
                    │Return Results  │      │
                    └─────────────────┘      │
                              │              │
                    ┌─────────▼──────────────▼──────┐
                    │ AI Response Generation        │
                    │ (process tool results)        │
                    └─────────────────────┬─────────┘
                                          │
                                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ SESSION UPDATE (session_manager.py)                             │
│ Add query + response to history                                 │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ HTTP RESPONSE (app.py)                                          │
│ {answer, sources, session_id} → JSON                           │
└──────────────────┬──────────────────────────────────────────────┘
                   │
                   ↓
┌─────────────────────────────────────────────────────────────────┐
│ FRONTEND (script.js)                                            │
│ Display response + sources + enable input                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Technologies

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Frontend** | HTML/CSS/JavaScript | User interface |
| **Server** | FastAPI | REST API endpoints |
| **Vector DB** | ChromaDB | Semantic search storage |
| **Embeddings** | SentenceTransformer | Convert text to vectors |
| **AI Model** | Claude (Anthropic API) | Generate intelligent responses |
| **Sessions** | In-memory manager | Maintain conversation context |

---

## File Architecture

```
├── frontend/
│   └── script.js          # User interface & HTTP requests
├── backend/
│   ├── app.py             # FastAPI routes & server setup
│   ├── rag_system.py      # RAG orchestrator
│   ├── ai_generator.py    # Claude API integration
│   ├── search_tools.py    # Tool definitions & execution
│   ├── vector_store.py    # ChromaDB interface
│   ├── session_manager.py # Conversation history
│   ├── document_processor.py  # Course document parsing
│   ├── models.py          # Data models
│   ├── config.py          # Configuration
│   └── [other utilities]
└── CLAUDE.md              # This file
```

---

## Query Flow Summary

1. **User Input**: Frontend captures query and session context
2. **API Request**: POST to `/api/query` with query + session_id
3. **RAG Routing**: Backend routes through RAGSystem orchestrator
4. **AI Decision**: Claude decides if semantic search is needed
5. **Tool Execution**: If search needed, execute vector search tool
6. **Vector Search**: ChromaDB performs semantic similarity matching
7. **Result Processing**: Format search results with metadata
8. **AI Response**: Claude synthesizes final response from search results
9. **Session Update**: Store query + response in conversation history
10. **Frontend Display**: Show response + sources, enable next query

This architecture ensures intelligent, context-aware responses with proper source attribution for course material queries.
