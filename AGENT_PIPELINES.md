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


## 6. Diagram Generation Pipeline

### Overview
When output format is "diagram", Khoj generates diagrams using either Mermaid.js (flowcharts, sequences, etc.) or Excalidraw (hand-drawn style).

### Pipeline Diagram

```
        User Diagram Request
                │
        ┌───────┴───────┐
        │               │
        ▼               ▼
   Mermaid.js      Excalidraw
   (Syntax-based)  (JSON-based)
        │               │
        ├───────────────┤
        │
        ▼
┌──────────────────────────┐
│  Phase 1: Improve        │
│  Description             │
└──────────────────────────┘
        │
┌───────┴────────┐
│  Mermaid:      │   Excalidraw:
│  improve_      │   improve_excalidraw_
│  mermaid_js_   │   diagram_description
│  diagram_      │
│  description   │
└───────┬────────┘
        │
   Input Context:
   - query
   - location
   - references (notes)
   - online_results
   - chat_history
        │
        ▼
   LLM Enhancement
   (Architect Role)
        │
┌───────┴────────┐
│ Mermaid:       │  Excalidraw:
│ Natural        │  Detailed layout with
│ language       │  primitives (Text,
│ description    │  Rectangle, Ellipse,
│                │  Line, Arrow)
└───────┬────────┘
        │
        ▼
┌──────────────────────────┐
│  Phase 2: Generate       │
│  Diagram Code            │
└──────────────────────────┘
        │
┌───────┴────────┐
│  Mermaid:      │   Excalidraw:
│  mermaid_js_   │   excalidraw_diagram_
│  diagram_      │   generation_prompt
│  generation    │
└───────┬────────┘
        │
   LLM Generation
   (Designer Role)
        │
┌───────┴────────────────────┐
│ Mermaid:                   │  Excalidraw:
│ flowchart TB               │  JSON with elements[]
│   A["Start"] --> B["End"]  │  [{type, x, y, id,
│                            │    label, ...}]
└───────┬────────────────────┘
        │
        ▼
   Validation & Error Handling
        │
┌───────┴────────┐
│ Valid?         │
│ Yes → Return   │
│ No  → Fallback │
└───────┬────────┘
        │
        ▼
   Display Diagram or
   ASCII Fallback
```

### Mermaid.js Flow Example

```
User: "Create a diagram showing the software development lifecycle"

Phase 1 - Improve Description:
  "Create a flowchart with 5 stages connected in sequence:
   Plan → Design → Implement → Test → Deploy.
   Add feedback loops from Test back to Design and from Deploy
   back to Plan for iterative development."

Phase 2 - Generate Syntax:
  flowchart TB
      plan["Plan"] --> design["Design"]
      design --> implement["Implement"]
      implement --> test["Test"]
      test --> deploy["Deploy"]
      test --> design
      deploy --> plan
```

### Excalidraw Flow Example

```
User: "Draw a circular development process"

Phase 1 - Improve Description:
  "Create a diagram with 3 ellipses labeled Design, Implement,
   and Feedback arranged in a circle. Connect them with arrows
   forming a circular flow: Design → Implement → Feedback → Design."

Phase 2 - Generate JSON:
  {
    "scratchpad": "Circular process with 3 stages...",
    "elements": [
      {"type": "ellipse", "x": -169, "y": 113, "id": "design",
       "label": {"text": "Design"}},
      {"type": "ellipse", "x": 62, "y": 394, "id": "implement",
       "label": {"text": "Implement"}},
      {"type": "arrow", "x": 21, "y": 273, "id": "arrow1",
       "start": {"id": "design"}, "end": {"id": "implement"}},
      ...
    ]
  }
```

### Prompts Used

**Mermaid.js Path:**
1. **improve_mermaid_js_diagram_description_prompt**
   - **Location**: `src/khoj/processor/conversation/prompts.py:311`
2. **mermaid_js_diagram_generation_prompt**
   - **Location**: `src/khoj/processor/conversation/prompts.py:348`

**Excalidraw Path:**
1. **improve_excalidraw_diagram_description_prompt**
   - **Location**: `src/khoj/processor/conversation/prompts.py:167`
2. **excalidraw_diagram_generation_prompt**
   - **Location**: `src/khoj/processor/conversation/prompts.py:205`

**Fallback:**
- **failed_diagram_generation** (ASCII fallback)
  - **Location**: `src/khoj/processor/conversation/prompts.py:425`

### Diagram Types Supported

**Mermaid.js:**
- Flowcharts
- Sequence Diagrams
- State Diagrams
- Gantt Charts
- Pie Charts

**Excalidraw:**
- Text
- Rectangles
- Ellipses
- Lines
- Arrows
- Hand-drawn aesthetic

---

## 7. Code Execution Pipeline

### Overview
When "code" is selected or the user requests data analysis/calculations, Khoj generates and executes Python code in a sandboxed environment.

### Pipeline Diagram

```
        User Request (code/analysis)
                    │
                    ▼
        ┌────────────────────────┐
        │  python_code_          │
        │  generation_prompt     │
        └────────────────────────┘
                    │
        ┌───────────┴──────────┐
        │  Input:              │
        │  - instructions      │
        │  - context (notes,   │
        │    online results)   │
        │  - chat_history      │
        │  - has_network       │
        │    _access           │
        │  - current_date      │
        └───────────┬──────────┘
                    │
                    ▼
           LLM Code Generation
           (Senior Engineer Role)
                    │
        ┌───────────┴──────────────┐
        │  Python Code in          │
        │  Markdown Block:         │
        │  ```python               │
        │  # Code here             │
        │  import pandas as pd     │
        │  ...                     │
        │  # Save output to file   │
        │  ```                     │
        └───────────┬──────────────┘
                    │
                    ▼
           Extract Code from Markdown
                    │
        ┌───────────┴──────────────┐
        │                          │
        ▼                          ▼
   E2B Sandbox              Terrarium Sandbox
   (Full packages)          (Limited packages)
        │                          │
        │  Available:              │  Available:
        │  - requests              │  - matplotlib
        │  - matplotlib            │  - pandas
        │  - pandas                │  - numpy
        │  - numpy                 │  - scipy
        │  - scipy                 │  - bs5
        │  - bs4                   │  - sympy
        │  - sympy                 │
        │  - torch                 │  NOT Available:
        │  - plotly                │  - requests
        │  - rdkit                 │  - torch
        │  + more                  │  - tensorflow
        │                          │  - rdkit
        └───────────┬──────────────┘
                    │
                    ▼
        Execute Code in Sandbox
                    │
        ┌───────────┴──────────────┐
        │  Capture:                │
        │  - stdout                │
        │  - stderr                │
        │  - Generated files       │
        │  - Execution time        │
        │  - Errors                │
        └───────────┬──────────────┘
                    │
                    ▼
        ┌────────────────────────┐
        │  code_executed_        │
        │  context               │
        └────────────────────────┘
                    │
        ┌───────────┴──────────┐
        │  Format Results:     │
        │  - Text output       │
        │  - File URLs         │
        │  - Error messages    │
        └───────────┬──────────┘
                    │
                    ▼
        Add to LLM Context
                    │
                    ▼
        LLM Interprets Results &
        Responds to User
```

### Code Generation Example

```
User: "Calculate compound interest on $10,000 at 5% for 3 years"

Generated Code:
```python
principal = 10000
rate = 0.05
years = 3

# Calculate compound interest
final_amount = principal * (1 + rate) ** years
interest_earned = final_amount - principal

print(f"Principal: ${principal:,.2f}")
print(f"Interest Rate: {rate * 100}%")
print(f"Time Period: {years} years")
print(f"Final Amount: ${final_amount:,.2f}")
print(f"Interest Earned: ${interest_earned:,.2f}")
```

Execution Output:
  Principal: $10,000.00
  Interest Rate: 5.0%
  Time Period: 3 years
  Final Amount: $11,576.25
  Interest Earned: $1,576.25
```

### Visualization Example

```
User: "Plot the function y = x^2 for x from -10 to 10"

Generated Code:
```python
import matplotlib.pyplot as plt
import numpy as np

x = np.linspace(-10, 10, 100)
y = x ** 2

plt.figure(figsize=(10, 6))
plt.plot(x, y, 'b-', linewidth=2)
plt.xlabel('x')
plt.ylabel('y = x²')
plt.title('Graph of y = x²')
plt.grid(True, alpha=0.3)
plt.savefig('quadratic_function.png', dpi=150)
```

Output:
  Generated file: quadratic_function.png (saved to sandbox)
  File URL returned to user
```

### Prompts Used

1. **python_code_generation_prompt**
   - **Purpose**: Generate secure, self-contained Python code
   - **Location**: `src/khoj/processor/conversation/prompts.py:878`

2. **code_executed_context**
   - **Purpose**: Format execution results for LLM
   - **Location**: `src/khoj/processor/conversation/prompts.py:1003`

3. **e2b_sandbox_context** / **terrarium_sandbox_context**
   - **Purpose**: Specify available packages
   - **Location**: `src/khoj/processor/conversation/prompts.py:1013-1019`

### Security Features

- Sandboxed execution (no host access)
- Limited package availability
- Execution timeout
- No persistent state between runs
- Output size limits

### Key Files

- `src/khoj/processor/tools/run_code.py:51` - Code execution engine

---

## 8. Research Agent Pipeline (/research mode)

### Overview
The research agent is a multi-iteration planning system that breaks down complex queries into sub-tasks and uses various tools to gather information.

### Pipeline Diagram

```
    User Query with /research Command
                    │
                    ▼
        ┌────────────────────────┐
        │  Initialize Research   │
        │  Agent                 │
        └────────────────────────┘
                    │
        ┌───────────┴──────────┐
        │  plan_function_      │
        │  execution prompt    │
        └───────────┬──────────┘
                    │
    ┌───────────────────────────────┐
    │   ITERATION LOOP              │
    │   (max 10 iterations)         │
    │                               │
    │       Current State           │
    │            │                  │
    │            ▼                  │
    │   ┌──────────────────┐       │
    │   │  Agent Decides   │       │
    │   │  Next Action     │       │
    │   └──────────────────┘       │
    │            │                  │
    │   ┌────────┴────────┐        │
    │   │ Available Tools:│        │
    │   │ - notes search  │        │
    │   │ - online search │        │
    │   │ - read webpage  │        │
    │   │ - run code      │        │
    │   │ - operate browser│       │
    │   └────────┬────────┘        │
    │            │                  │
    │            ▼                  │
    │   Execute Selected Tool      │
    │            │                  │
    │   ┌────────┴────────┐        │
    │   │  Tool Result    │        │
    │   │  Added to       │        │
    │   │  Context        │        │
    │   └────────┬────────┘        │
    │            │                  │
    │            ▼                  │
    │   Agent Evaluates Progress   │
    │            │                  │
    │   ┌────────┴────────┐        │
    │   │ Task Complete?  │        │
    │   │ Yes → Exit      │        │
    │   │ No  → Continue  │        │
    │   └────────┬────────┘        │
    │            │                  │
    │            └──────┐           │
    │                   │           │
    └───────────────────┘           │
                        │           │
                        ▼           │
            Repeat Iteration  ←─────┘
                        │
                        ▼
        ┌────────────────────────┐
        │  Compile All Results   │
        └────────────────────────┘
                        │
                        ▼
        Final Response to User
```

### Research Flow Example

```
User: "Find the population of my hometown and compare it to neighboring cities"

Iteration 1:
  Agent Decision: Use notes search to find user's hometown
  Tool: notes_search
  Query: "hometown birth certificate childhood"
  Result: Found "Born in Springfield, MA"
  
Iteration 2:
  Agent Decision: Search online for Springfield MA population
  Tool: online_search
  Query: "Springfield Massachusetts population 2025"
  Result: "Population: 155,929 (2024 estimate)"

Iteration 3:
  Agent Decision: Find neighboring cities
  Tool: online_search
  Query: "cities near Springfield Massachusetts"
  Result: "Chicopee, Holyoke, West Springfield, Agawam"

Iteration 4:
  Agent Decision: Get populations of neighboring cities
  Tool: run_code
  Code: Web scraping to get population data for each city
  Result: 
    Chicopee: 55,560
    Holyoke: 38,247
    West Springfield: 28,835
    Agawam: 28,692

Iteration 5:
  Agent Decision: Task complete, compile results
  Result: Comprehensive comparison with all data
```

### Tool Selection Logic

```
Available Tools:
├─ notes_search        → Semantic search in user's documents
├─ notes_search_regex  → Regex pattern matching in documents
├─ view_file          → Read specific file contents
├─ list_files         → List files in directory
├─ online_search      → Web search
├─ read_webpage       → Read specific webpage
├─ run_code           → Execute Python for data processing
└─ operate_browser    → Browser automation for complex tasks

Agent chooses tools based on:
- Previous results
- Task requirements
- Available data sources
- Complexity of subtask
```

### Prompt Used

- **plan_function_execution**
  - **Purpose**: Multi-step planning and iteration
  - **Location**: `src/khoj/processor/conversation/prompts.py:644`
  - **Key Features**:
    - Self-contained requests to tool AIs
    - Independent, sequential steps
    - Creative strategies when previous attempts fail
    - No user confirmation for information gathering
    - Maximum iterations limit

### Data Flow

```
User Query
    ↓
plan_function_execution (initial)
    ↓
Agent decides: "Search user's notes for hometown"
    ↓
notes_search tool executed
    ↓
Results added to context
    ↓
plan_function_execution (iteration 2)
    ↓
Agent decides: "Search online for population data"
    ↓
online_search tool executed
    ↓
Results added to context
    ↓
...continue iterations...
    ↓
Agent decides: "Task complete, summarize findings"
    ↓
Final response compiled and returned
```

### Key Files

- `src/khoj/routers/research.py:220` - Research agent implementation
- `src/khoj/processor/conversation/prompts.py:644` - Planning prompt

---


## 9. Operator Agent Pipeline (/operator mode)

### Overview
The operator agent controls a web browser or computer desktop using vision-enabled LLMs. It can click, type, navigate, and extract information from visual interfaces.

### Pipeline Diagram

```
    User Request with /operator Command
                    │
                    ▼
        ┌────────────────────────┐
        │  Initialize Operator   │
        │  Environment           │
        └────────────────────────┘
                    │
        ┌───────────┴──────────┐
        │  Environment Type:   │
        │  - Browser (Chromium)│
        │  - Computer (Desktop)│
        └───────────┬──────────┘
                    │
    ┌───────────────────────────────────┐
    │   AGENT ACTION LOOP               │
    │   (max 100 iterations)            │
    │                                   │
    │   ┌──────────────────────┐       │
    │   │  1. Take Screenshot  │       │
    │   └──────────────────────┘       │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────┐       │
    │   │  2. Encode Image     │       │
    │   │     (Base64)         │       │
    │   └──────────────────────┘       │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────────┐   │
    │   │  3. Vision LLM Analysis  │   │
    │   │     (Claude/GPT-4V)      │   │
    │   └──────────────────────────┘   │
    │            │                      │
    │   ┌────────┴────────────┐        │
    │   │  System Prompt:     │        │
    │   │  - Browser/Computer │        │
    │   │    instructions     │        │
    │   │  - Current URL      │        │
    │   │  - Date context     │        │
    │   │  - Max iterations   │        │
    │   └────────┬────────────┘        │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────┐       │
    │   │  4. LLM Decides      │       │
    │   │     Action(s)        │       │
    │   └──────────────────────┘       │
    │            │                      │
    │   ┌────────┴────────────┐        │
    │   │  Available Actions: │        │
    │   │  - mouse_move       │        │
    │   │  - left_click       │        │
    │   │  - right_click      │        │
    │   │  - double_click     │        │
    │   │  - type             │        │
    │   │  - key              │        │
    │   │  - scroll           │        │
    │   │  - goto (browser)   │        │
    │   │  - back (browser)   │        │
    │   │  - screenshot       │        │
    │   │  - wait             │        │
    │   └────────┬────────────┘        │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────┐       │
    │   │  5. Execute Action   │       │
    │   │     in Environment   │       │
    │   └──────────────────────┘       │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────┐       │
    │   │  6. Capture Result   │       │
    │   │     (Success/Error)  │       │
    │   └──────────────────────┘       │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────┐       │
    │   │  7. Add to Message   │       │
    │   │     History          │       │
    │   └──────────────────────┘       │
    │            │                      │
    │            ▼                      │
    │   ┌──────────────────────┐       │
    │   │  Task Complete?      │       │
    │   │  Yes → Exit          │       │
    │   │  No  → Continue      │       │
    │   └──────────┬───────────┘       │
    │              │                    │
    │              └──────┐             │
    │                     │             │
    └─────────────────────┘             │
                          │             │
                  Next Iteration  ←─────┘
                          │
                          ▼
        ┌────────────────────────────┐
        │  operator_execution_       │
        │  context                   │
        └────────────────────────────┘
                          │
                          ▼
        Format Results for User
                          │
                          ▼
        Final Response with
        Screenshots & Actions Taken
```

### Action Sequence Example

```
User: "Search for 'Khoj AI' on DuckDuckGo and summarize the first result"

Iteration 1:
  Screenshot: Blank browser
  LLM Decision: goto("https://duckduckgo.com")
  Action: Navigate to DuckDuckGo
  Result: Success

Iteration 2:
  Screenshot: DuckDuckGo homepage
  LLM Decision: left_click(x=500, y=300) on search box
  Action: Click search input
  Result: Search box focused

Iteration 3:
  Screenshot: Search box focused
  LLM Decision: type("Khoj AI")
  Action: Type search query
  Result: Text entered

Iteration 4:
  Screenshot: Query typed
  LLM Decision: key("Return")
  Action: Press Enter
  Result: Search submitted

Iteration 5:
  Screenshot: Search results page
  LLM Decision: left_click(x=400, y=200) on first result
  Action: Click first search result
  Result: Navigated to page

Iteration 6:
  Screenshot: Article page
  LLM Decision: scroll(direction="down", amount=5)
  Action: Scroll to read content
  Result: Page scrolled

Iteration 7:
  Screenshot: More content visible
  LLM Analysis: Extract key information from visible text
  Result: Summary compiled

Iteration 8:
  LLM Decision: Task complete
  Result: Return summary to user
```

### Browser vs Computer Mode

**Browser Mode:**
- Chromium browser via Playwright
- Limited to web interactions
- No filesystem access
- Helper functions: `goto()`, `back()`
- DuckDuckGo for searches

**Computer Mode:**
- Full desktop control
- Can interact with any application
- File access available
- More general-purpose
- Same action primitives

### Prompts Used

1. **Browser Operator Instructions**
   - **Purpose**: System prompt for browser automation
   - **Location**: `src/khoj/processor/operator/operator_agent_anthropic.py:554`
   - **Key Instructions**:
     - Operate Chromium browser with Playwright
     - Use DuckDuckGo for searches
     - Zoom out to see full pages
     - Chain multiple actions when possible
     - Maximum iterations limit

2. **Computer Operator Instructions**
   - **Purpose**: System prompt for desktop automation
   - **Location**: `src/khoj/processor/operator/operator_agent_anthropic.py:577`
   - **Key Instructions**:
     - Full computer interaction
     - Document/webpage handling
     - Action chaining
     - Iteration limits

3. **operator_execution_context**
   - **Purpose**: Format operator results for user
   - **Location**: `src/khoj/processor/conversation/prompts.py:1021`

### Vision-Action Loop

```
Screenshot (PNG/Base64)
    ↓
Vision LLM (Claude 3.5 Sonnet / GPT-4V)
    ↓
Analyze UI elements, text, buttons, forms
    ↓
Decide action(s) with coordinates
    ↓
Execute in environment
    ↓
Capture new screenshot
    ↓
Repeat until task complete
```

### Key Files

- `src/khoj/processor/operator/__init__.py:34` - Main operator function
- `src/khoj/processor/operator/operator_agent_anthropic.py` - Anthropic implementation
- `src/khoj/processor/operator/operator_agent_openai.py` - OpenAI implementation

---

## 10. Automation & Scheduled Tasks

### Overview
Users can schedule recurring tasks that run automatically. The system parses natural language schedules, executes tasks on cron schedules, and optionally notifies users of results.

### Pipeline Diagram

```
    User: "Show me top HN posts every Monday at 6pm"
                        │
                        ▼
        ┌────────────────────────────┐
        │  crontime_prompt           │
        └────────────────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │  Input:                      │
        │  - query (user request)      │
        │  - chat_history              │
        └───────────────┬──────────────┘
                        │
                        ▼
               LLM Parsing
          (Schedule + Query + Subject)
                        │
        ┌───────────────┴──────────────────┐
        │  Output JSON:                    │
        │  {                               │
        │    "crontime": "0 18 * * 1",     │
        │    "query": "/automated_task     │
        │             Top HN posts",       │
        │    "subject": "Weekly Top        │
        │                HN Posts"         │
        │  }                               │
        └───────────────┬──────────────────┘
                        │
                        ▼
        ┌────────────────────────────┐
        │  Create Automation Job     │
        │  (APScheduler)             │
        └────────────────────────────┘
                        │
                        ▼
        Store in Database with User Timezone
                        │
                        ▼
        ═══════════════════════════════
        SCHEDULED EXECUTION (Recurring)
        ═══════════════════════════════
                        │
                        ▼
        Cron Trigger Fires
                        │
                        ▼
        ┌────────────────────────────┐
        │  Execute Automated Query   │
        │  (Run through chat API)    │
        └────────────────────────────┘
                        │
                        ▼
        Generate AI Response
        (Uses full chat pipeline)
                        │
                        ▼
        ┌────────────────────────────┐
        │  to_notify_or_not          │
        │  (Notification Decision)   │
        └────────────────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │  Input:                      │
        │  - original_query            │
        │  - executed_query            │
        │  - response                  │
        └───────────────┬──────────────┘
                        │
                        ▼
               LLM Decision
                        │
        ┌───────────────┴──────────────────┐
        │  Output JSON:                    │
        │  {                               │
        │    "reason": "AI generated        │
        │               result",            │
        │    "decision": "Yes" or "No"     │
        │  }                               │
        └───────────────┬──────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │  Decision == "Yes"?          │
        │  Yes → Send Notification     │
        │  No  → Skip (wait for next)  │
        └───────────────┬──────────────┘
                        │
                        ▼
        ┌────────────────────────────┐
        │  automation_format_prompt  │
        │  (Format for Email)        │
        └────────────────────────────┘
                        │
        ┌───────────────┴──────────────┐
        │  Input:                      │
        │  - username                  │
        │  - original_query            │
        │  - executed_query            │
        │  - response                  │
        └───────────────┬──────────────┘
                        │
                        ▼
        LLM Formatting
        (Markdown report with sections)
                        │
                        ▼
        ┌────────────────────────────┐
        │  Send Email Notification   │
        │  Subject: Task subject     │
        │  Body: Formatted response  │
        └────────────────────────────┘
                        │
                        ▼
        User Receives Email
                        │
                        ▼
        Wait for Next Cron Trigger
```

### Schedule Parsing Examples

```
User: "Show a Calvin and Hobbes quote every morning at 9am"
Parsed:
  crontime: "0 9 * * *"     (9am daily)
  query: "Share a funny Calvin and Hobbes quote from my notes"
  subject: "Your Daily Calvin and Hobbes Quote"

User: "Notify me when Khoj 2.0.0 is released"
Parsed:
  crontime: "0 10 * * *"    (10am daily, checks condition)
  query: "/automated_task /research What is the latest Khoj version?"
  subject: "Khoj Version 2.0.0 Release"

User: "Tech news on first Sunday of every month"
Parsed:
  crontime: "0 8 1-7 * 0"   (8am, days 1-7, Sunday)
  query: "/automated_task Find latest tech news"
  subject: "Your Monthly Tech News"
```

### Notification Decision Logic

```
Example 1 - Notify:
  Original: "Show a quote every morning"
  Executed: "Share a Calvin and Hobbes quote"
  Response: "Here's a quote: 'It's not denial...'"
  Decision: "Yes" (content generated successfully)
  → Email sent

Example 2 - Don't Notify:
  Original: "Notify when it rains tomorrow"
  Executed: "Will it rain tomorrow?"
  Response: "Tomorrow will be sunny"
  Decision: "No" (condition not met)
  → Email not sent, wait for next day

Example 3 - Notify:
  Original: "Summary of completed tasks daily"
  Executed: "Generate task summary"
  Response: "You completed: 1. Meeting, 2. Report"
  Decision: "Yes" (summary available)
  → Email sent
```

### Prompts Used

1. **crontime_prompt**
   - **Purpose**: Parse natural language into cron schedule
   - **Location**: `src/khoj/processor/conversation/prompts.py:1033`

2. **subject_generation**
   - **Purpose**: Generate email subject for task
   - **Location**: `src/khoj/processor/conversation/prompts.py:1099`

3. **to_notify_or_not**
   - **Purpose**: Decide if user should be notified
   - **Location**: `src/khoj/processor/conversation/prompts.py:1214`

4. **automation_format_prompt**
   - **Purpose**: Format response as email-ready markdown
   - **Location**: `src/khoj/processor/conversation/prompts.py:1255`

### Cron Time Format

```
 ┌─────── minute (0-59)
 │ ┌────── hour (0-23)
 │ │ ┌───── day of month (1-31)
 │ │ │ ┌──── month (1-12)
 │ │ │ │ ┌─── day of week (0-7, Sunday=0 or 7)
 │ │ │ │ │
 * * * * *

Examples:
  "0 9 * * *"     - 9am every day
  "0 18 * * 1"    - 6pm every Monday
  "0 8 1-7 * 0"   - 8am first Sunday of month
  "0 */4 * * *"   - Every 4 hours
```

### Key Files

- `src/khoj/routers/api_automation.py:51` - Automation endpoints
- `src/khoj/processor/conversation/prompts.py:1033` - Schedule parsing

---

## Summary

This document has covered 10 major agent pipelines in Khoj:

1. **Main Chat Pipeline** - Core conversation flow with intelligent routing
2. **Data Source Selection** - LLM-driven decision on which sources to use
3. **Query Generation & Document Search** - Semantic search in user's notes
4. **Online Search Pipeline** - Web search and webpage reading
5. **Image Generation** - AI-powered image creation with enhancement
6. **Diagram Generation** - Mermaid.js and Excalidraw diagrams
7. **Code Execution** - Sandboxed Python code generation and execution
8. **Research Agent** - Multi-iteration planning and tool usage
9. **Operator Agent** - Vision-guided browser/desktop automation
10. **Automation & Scheduled Tasks** - Cron-based recurring AI tasks

### Key Architectural Patterns

**Multi-Stage Prompting:**
- Most complex pipelines use 2-4 sequential prompts
- Each prompt refines the output of the previous
- Structured JSON responses enable deterministic parsing

**Parallel Execution:**
- Multiple data sources gathered concurrently
- Document searches across file types run in parallel
- Streaming responses for reduced latency

**Context Management:**
- Token-aware message building
- Selective truncation of large results
- Per-user result caching

**Error Handling:**
- Graceful fallbacks at each stage
- Partial state preservation
- ASCII diagram fallback for generation failures

**Security:**
- Sandboxed code execution
- Browser automation isolation
- Prompt safety validation

### Prompt Orchestration

All prompts are centrally defined in `src/khoj/processor/conversation/prompts.py` and dynamically injected with:
- User context (name, location, timezone)
- Temporal context (date, day of week)
- Conversation history
- Retrieved data (notes, web results)
- Agent personality

This architecture enables Khoj to provide intelligent, context-aware assistance across a wide range of tasks while maintaining security and reliability.

---

**End of Agent Pipelines Documentation**
