# Khoj Agent Pipelines & Workflows Documentation

This document provides comprehensive documentation of all agent workflows and execution pipelines in Khoj, including sequence diagrams, data flow, and prompt usage.

## Table of Contents

1. [Main Chat Pipeline](#1-main-chat-pipeline)
2. [Data Source Selection](#2-data-source-selection)
3. [Query Generation & Document Search](#3-query-generation--document-search)
4. [Online Search Pipeline](#4-online-search-pipeline)
5. [Image Generation Pipeline](#5-image-generation-pipeline)
6. [Diagram Generation Pipeline](#6-diagram-generation-pipeline)
7. [Code Execution Pipeline](#7-code-execution-pipeline)
8. [Research Agent Pipeline](#8-research-agent-pipeline)
9. [Operator Agent Pipeline](#9-operator-agent-pipeline)
10. [Automation & Scheduled Tasks](#10-automation--scheduled-tasks)

---

## 1. Main Chat Pipeline

### Overview
The main chat pipeline is the default conversation flow that handles most user interactions. It intelligently routes queries to appropriate data sources and generates responses.

### Entry Point
- **API Endpoint**: `POST /api/chat` or `WebSocket /api/chat/ws`
- **File**: `src/khoj/routers/api_chat.py:665`
- **Trigger**: User submits a query through any Khoj client

### Pipeline Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                         MAIN CHAT PIPELINE                             │
└────────────────────────────────────────────────────────────────────────┘

    User Query
        │
        ▼
   ┌─────────┐
   │ Phase 1 │  Query Validation & Setup
   └─────────┘
        │
        ├─→ Validate query (not empty)
        ├─→ Get/Create conversation
        ├─→ Extract command (/notes, /online, /research, etc.)
        │
        ▼
   ┌─────────┐
   │ Phase 2 │  Data Source & Output Format Selection
   └─────────┘
        │
        ├─→ Prompt: pick_relevant_tools
        │   Input:  {query, chat_history, sources, outputs}
        │   Output: {"source": ["notes", "online"], "output": "text"}
        │
        ▼
   ┌─────────┐
   │ Phase 3 │  Context Gathering (Parallel)
   └─────────┘
        │
        ├─────────────┬──────────────┬─────────────┬──────────────┐
        │             │              │             │              │
        ▼             ▼              ▼             ▼              ▼
   Notes Search  Online Search  Webpage Read  Code Exec  Operator Mode
        │             │              │             │              │
        │         See §4           See §4        See §7         See §9
        │             │              │             │              │
        ▼             │              │             │              │
  Query Generation    │              │             │              │
  (extract_questions) │              │             │              │
        │             │              │             │              │
        ▼             │              │             │              │
  Document Search     │              │             │              │
  (semantic search)   │              │             │              │
        │             │              │             │              │
        └─────────────┴──────────────┴─────────────┴──────────────┘
                              │
                              ▼
                    Compiled Context
                    {references, online_results,
                     code_results, operator_results}
                              │
                              ▼
                      ┌─────────┐
                      │ Phase 4 │  Message Building
                      └─────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   System Message      Context Message      Chat History
   (personality)       (references)         (previous turns)
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
                              ▼
                    Complete Message Array
                              │
                              ▼
                      ┌─────────┐
                      │ Phase 5 │  LLM Generation
                      └─────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   OpenAI Provider    Anthropic Provider    Google Provider
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
                              ▼
                    Streaming Response Tokens
                              │
                              ▼
                      ┌─────────┐
                      │ Phase 6 │  Storage & Streaming
                      └─────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
   Stream to User    Save to Database    Store References
        │                     │                     │
        └─────────────────────┴─────────────────────┘
                              │
                              ▼
                      Response Complete
```

### Prompts Used

1. **pick_relevant_tools** (Phase 2)
   - **Purpose**: Determines which data sources and output format to use
   - **Input**: User query, chat history, available sources/outputs
   - **Output**: JSON with source array and output format
   - **Location**: `src/khoj/processor/conversation/prompts.py:691`

2. **personality** (Phase 4)
   - **Purpose**: Defines Khoj's personality and capabilities
   - **Input**: Current date, day of week
   - **Output**: System message text
   - **Location**: `src/khoj/processor/conversation/prompts.py:5`

3. **notes_conversation** (Phase 4, if notes selected)
   - **Purpose**: Instructions for using personal notes
   - **Input**: References from document search
   - **Output**: Formatted context message
   - **Location**: `src/khoj/processor/conversation/prompts.py:92`

4. **online_search_conversation** (Phase 4, if online selected)
   - **Purpose**: Instructions for using web search results
   - **Input**: Online search results
   - **Output**: Formatted context message
   - **Location**: `src/khoj/processor/conversation/prompts.py:438`

### Data Transformations

```
User Query (string)
    ↓
Data Source Selection (LLM inference)
    ↓
Context Objects {references[], online_results{}, code_results{}}
    ↓
Formatted Messages (ChatMessage[])
    ↓
LLM Response (streaming tokens)
    ↓
Saved Conversation (database)
```

### Key Files

- `src/khoj/routers/api_chat.py` - Main chat endpoint
- `src/khoj/routers/helpers.py` - Context gathering and message building
- `src/khoj/processor/conversation/utils.py` - LLM provider interfaces

---


## 2. Data Source Selection

### Overview
Before gathering context, Khoj uses an LLM to intelligently decide which data sources to query based on the user's question and available resources.

### Pipeline Diagram

```
                    User Query + Chat History
                              │
                              ▼
                ┌──────────────────────────┐
                │  pick_relevant_tools     │
                │  (LLM Decision)          │
                └──────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │    Input Variables:            │
              │    - chat_history              │
              │    - query                     │
              │    - sources (available)       │
              │    - outputs (available)       │
              └───────────────┬───────────────┘
                              │
                              ▼
                    LLM Analysis & Decision
                              │
                              ▼
              ┌───────────────┴───────────────┐
              │    JSON Response:              │
              │    {                           │
              │      "source": ["notes",       │
              │                  "online"],    │
              │      "output": "text"          │
              │    }                           │
              └───────────────┬───────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
        Sources Selected               Output Format
              │                               │
    ┌─────────┴─────────┐                    │
    │                   │                    │
    ▼                   ▼                    ▼
  notes              online                text
  webpage            code               or image
  operator           general            or diagram
```

### Decision Factors

```
Available Sources:
├─ notes:     User has synced documents
├─ online:    Web search enabled
├─ webpage:   Web reading enabled  
├─ code:      Code sandbox enabled
├─ operator:  Browser automation enabled
└─ general:   Always available (no external data)

Output Formats:
├─ text:      Standard response
├─ image:     Generate image with DALL-E/Gemini
└─ diagram:   Generate Excalidraw or Mermaid diagram
```

### Prompt Used
- **pick_relevant_tools** (`src/khoj/processor/conversation/prompts.py:691`)

### Key Files
- `src/khoj/routers/helpers.py:347` - Implementation

---

## 3. Query Generation & Document Search

### Overview
When "notes" is selected as a data source, Khoj generates semantic search queries and searches the user's knowledge base.

### Pipeline Diagram

```
              User Query + Chat History
                        │
                        ▼
            ┌────────────────────────┐
            │  extract_questions     │
            │  (Query Generation)    │
            └────────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │   Input:                      │
        │   - text (user query)         │
        │   - chat_history              │
        │   - location                  │
        │   - username                  │
        │   - current_date              │
        │   - max_queries (1-5)         │
        └───────────────┬───────────────┘
                        │
                        ▼
              LLM Query Decomposition
                        │
        ┌───────────────┴───────────────┐
        │   Output JSON:                │
        │   {                           │
        │     "queries": [              │
        │       "semantic query 1",     │
        │       "query 2 dt>=date",     │
        │       "location-based query"  │
        │     ]                         │
        │   }                           │
        └───────────────┬───────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  For Each Query  │
              └──────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │                               │
        ▼                               ▼
  Embedding Generation         Apply Filters
  (sentence-transformers)      (date, file type)
        │                               │
        └───────────────┬───────────────┘
                        │
                        ▼
            ┌────────────────────────┐
            │  Vector Search         │
            │  (Semantic Similarity) │
            └────────────────────────┘
                        │
            ┌───────────┴───────────┐
            │   Search across:      │
            │   - Markdown files    │
            │   - PDF documents     │
            │   - Org files         │
            │   - Plain text        │
            │   - Images (OCR)      │
            │   - GitHub repos      │
            │   - Notion pages      │
            └───────────┬───────────┘
                        │
                        ▼
              Rank & Deduplicate Results
                        │
                        ▼
        ┌───────────────────────────────┐
        │   Compiled References:        │
        │   [                           │
        │     {                         │
        │       "query": "...",         │
        │       "compiled": "text",     │
        │       "file": "note.md",      │
        │       "uri": "file://..."     │
        │     }                         │
        │   ]                           │
        └───────────────┬───────────────┘
                        │
                        ▼
              Format as YAML Context
                        │
                        ▼
              Add to notes_conversation
                   Prompt Template
```

### Query Generation Patterns

The extract_questions prompt generates queries using diverse perspectives:

```
Example Input:
  "How was my trip to Cambodia?"

Generated Queries:
  1. "How was my trip to Cambodia?"       # Direct question
  2. "Angkor Wat temple visit"            # Specific location
  3. "Flight to Phnom Penh"               # Travel details
  4. "Expenses in Cambodia"               # Financial aspect
  5. "Stay in Cambodia"                   # Accommodation
```

### Date Filtering

Queries can include temporal filters using `dt>=` and `dt<` operators:

```
Example:
  "What did I do last Christmas?"

Generated Query:
  "What did I do for Christmas 2024 dt>='2024-12-25' dt<'2024-12-26'"
```

### Prompts Used

1. **extract_questions_system_prompt**
   - **Purpose**: Generate 1-5 semantic search queries
   - **Location**: `src/khoj/processor/conversation/prompts.py:522`

2. **extract_questions_user_message**
   - **Purpose**: Format user query and chat history
   - **Location**: `src/khoj/processor/conversation/prompts.py:569`

3. **notes_conversation**
   - **Purpose**: Format retrieved references for LLM
   - **Location**: `src/khoj/processor/conversation/prompts.py:92`

### Data Flow

```
"What projects did I work on last month?"
    ↓
extract_questions (LLM)
    ↓
["Projects I worked on in January 2025 dt>='2025-01-01' dt<'2025-02-01'",
 "Work accomplishments January",
 "Code commits and PRs January 2025"]
    ↓
Embedding Generation (for each query)
    ↓
Vector Search (across all document types)
    ↓
[{file: "work_log_jan2025.md", snippet: "..."},
 {file: "github_prs.md", snippet: "..."}]
    ↓
Deduplication & Ranking
    ↓
YAML-formatted References
    ↓
Added to LLM Context
```

### Key Files

- `src/khoj/routers/helpers.py:1273` - extract_questions()
- `src/khoj/routers/helpers.py:1166` - search_documents()
- `src/khoj/processor/conversation/prompts.py:522` - Query generation prompt

---

## 4. Online Search Pipeline

### Overview
When "online" is selected, Khoj searches the web and optionally reads specific webpages to answer queries.

### Pipeline Diagram

```
                    User Query
                        │
            ┌───────────┴───────────┐
            │                       │
            ▼                       ▼
    Generate Subqueries      Infer Webpages
    (optional, for search)   (optional, for reading)
            │                       │
            ▼                       ▼
┌────────────────────────┐ ┌────────────────────────┐
│ online_search_         │ │ infer_webpages_        │
│ conversation_subqueries│ │ to_read                │
└────────────────────────┘ └────────────────────────┘
            │                       │
┌───────────┴────────┐  ┌───────────┴───────────┐
│ Input:             │  │ Input:                │
│ - query            │  │ - query               │
│ - chat_history     │  │ - chat_history        │
│ - max_queries (3)  │  │ - max_webpages (1)    │
└───────────┬────────┘  └───────────┬───────────┘
            │                       │
            ▼                       ▼
  LLM Query Generation    LLM URL Inference
            │                       │
┌───────────┴────────┐  ┌───────────┴───────────┐
│ Output JSON:       │  │ Output JSON:          │
│ {                  │  │ {                     │
│   "queries": [     │  │   "links": [          │
│     "query 1",     │  │     "https://url"     │
│     "query 2"      │  │   ]                   │
│   ]                │  │ }                     │
│ }                  │  │                       │
└───────────┬────────┘  └───────────┬───────────┘
            │                       │
            ▼                       ▼
┌────────────────────────┐ ┌────────────────────────┐
│  Web Search            │ │  Read Webpage          │
│  (Serper/SearXNG)      │ │  (Firecrawl/BS4)       │
└────────────────────────┘ └────────────────────────┘
            │                       │
            ▼                       ▼
┌────────────────────────┐ ┌────────────────────────┐
│  For Each Subquery:    │ │  For Each URL:         │
│  - Answer box          │ │  - Fetch HTML          │
│  - Search results      │ │  - Convert to Markdown │
│  - Snippets            │ │  - Extract relevant    │
│  - URLs                │ │    content             │
└────────────────────────┘ └────────────────────────┘
            │                       │
            └───────────┬───────────┘
                        │
                        ▼
            ┌────────────────────────┐
            │  Compiled Online       │
            │  Results (dict)        │
            └────────────────────────┘
                        │
        ┌───────────────┴───────────────┐
        │  Structure:                   │
        │  {                            │
        │    "query1": {                │
        │      "answerBox": {...},      │
        │      "webpages": [{           │
        │        "link": "...",         │
        │        "title": "...",        │
        │        "snippet": "..."       │
        │      }]                       │
        │    },                         │
        │    "webpages": {              │
        │      "url": "extracted text"  │
        │    }                          │
        │  }                            │
        └───────────────┬───────────────┘
                        │
                        ▼
            Format as YAML Context
                        │
                        ▼
        Add to online_search_conversation
               Prompt Template
```

### Search Query Generation Example

```
Input:
  "What's the latest news about Claude AI?"

Subqueries Generated:
  ["latest news Claude AI Anthropic",
   "Claude AI new features 2025",
   "site:anthropic.com Claude updates"]
```

### Webpage Reading Example

```
Input:
  "Summarize the top posts on Hacker News"

URL Inferred:
  ["https://news.ycombinator.com/best"]

Process:
  Fetch HTML → Extract articles → Convert to Markdown →
  Return formatted text
```

### Prompts Used

1. **online_search_conversation_subqueries**
   - **Purpose**: Generate Google search queries
   - **Location**: `src/khoj/processor/conversation/prompts.py:805`

2. **infer_webpages_to_read**
   - **Purpose**: Construct specific webpage URLs
   - **Location**: `src/khoj/processor/conversation/prompts.py:761`

3. **extract_relevant_information**
   - **Purpose**: Extract relevant content from webpages
   - **Location**: `src/khoj/processor/conversation/prompts.py:591`

4. **online_search_conversation**
   - **Purpose**: Format online results for LLM
   - **Location**: `src/khoj/processor/conversation/prompts.py:438`

### Key Files

- `src/khoj/processor/tools/online_search.py:69` - search_online()
- `src/khoj/processor/tools/online_search.py` - read_webpages()

---

## 5. Image Generation Pipeline

### Overview
When output format is "image", Khoj uses AI image generation models (DALL-E 3 or Gemini 2.0) to create images based on the user's description.

### Pipeline Diagram

```
            User Image Request
                    │
                    ▼
    ┌────────────────────────────────┐
    │  Enhance Image Description     │
    │  (enhance_image_system_message)│
    └────────────────────────────────┘
                    │
    ┌───────────────┴───────────────┐
    │  Input:                       │
    │  - User request               │
    │  - Location context           │
    │  - References (from notes)    │
    │  - Online results             │
    └───────────────┬───────────────┘
                    │
                    ▼
           LLM Enhancement
           (Media Artist Role)
                    │
    ┌───────────────┴───────────────┐
    │  Output JSON:                 │
    │  {                            │
    │    "description": "detailed   │
    │                    image      │
    │                    prompt",   │
    │    "shape": "square" or       │
    │             "portrait" or     │
    │             "landscape"       │
    │  }                            │
    └───────────────┬───────────────┘
                    │
            ┌───────┴────────┐
            │                │
            ▼                ▼
      DALL-E 3          Gemini 2.0
      (OpenAI)          (Google)
            │                │
            └────────┬───────┘
                     │
                     ▼
          ┌──────────────────┐
          │  Generated Image │
          │  (URL or base64) │
          └──────────────────┘
                     │
                     ▼
         Upload to Cloud Storage
                     │
                     ▼
    ┌────────────────────────────────┐
    │  generated_assets_context      │
    │  (Summary for user)            │
    └────────────────────────────────┘
                     │
                     ▼
         Return Image URL to User
```

### Enhancement Example

```
User Input:
  "Draw a sunset over mountains"

Enhanced Description (by LLM):
  "A breathtaking sunset over majestic mountain peaks. Golden hour 
  lighting with warm orange and pink hues painting the sky. Dramatic
  cloud formations catching the last rays of sunlight. Snow-capped
  mountain ridges in sharp relief against the colorful sky. 
  Foreground shows alpine meadow with wildflowers. Professional 
  landscape photography style, wide angle lens, f/8 aperture, 
  golden hour timing."

Shape Selected:
  "landscape"
```

### Prompt Transformations

The enhancement prompt converts negations to positive alternatives:

```
User: "Draw a beach with no people"
Enhanced: "Draw a solitary, empty beach with pristine sand"

User: "City at night without sun"
Enhanced: "City at night under moonlight and stars"
```

### Prompts Used

1. **enhance_image_system_message**
   - **Purpose**: Transform user request into detailed image prompt
   - **Location**: `src/khoj/processor/conversation/prompts.py:116`

2. **generated_assets_context**
   - **Purpose**: Provide summary after image creation
   - **Location**: `src/khoj/processor/conversation/prompts.py:151`

### Data Flow

```
"Paint a cyberpunk city"
    ↓
enhance_image_system_message (LLM)
    ↓
{description: "Neon-lit cyberpunk metropolis with towering 
              skyscrapers, holographic advertisements, 
              rain-slicked streets reflecting vibrant colors...",
 shape: "landscape"}
    ↓
DALL-E 3 / Gemini 2.0 API
    ↓
Image Binary Data
    ↓
Upload to Cloud Storage
    ↓
Public Image URL
    ↓
Display to User with Summary
```

### Key Files

- `src/khoj/processor/image/generate.py:39` - generate_image()
- `src/khoj/processor/conversation/prompts.py:116` - Enhancement prompt

---

