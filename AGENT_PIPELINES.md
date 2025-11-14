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

