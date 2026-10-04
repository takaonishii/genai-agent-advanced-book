# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

All commands should be run from `chapter4/`.

```bash
uv sync                        # Install dependencies
source .venv/bin/activate      # Activate virtual environment
```

Create `chapter4/.env`:
```env
OPENAI_API_KEY=your_openai_api_key
OPENAI_API_BASE="https://api.openai.com/v1"
OPENAI_MODEL="gpt-4o-2024-08-06"
```

`src/configs.py` reads from `./chapter4/.env` via pydantic-settings (path is relative to repo root, not chapter4). `src/scripts/create_index.py` reads from `.env` directly (relative to chapter4).

## Commands

```bash
# Search engine (Elasticsearch + Qdrant via Docker)
make start.engine    # Start containers
make create.index    # Build search indexes (loads PDFs + CSV into ES and Qdrant)
make delete.index    # Delete indexes
make stop.engine     # Stop containers

# Linting
uv run ruff check .
uv run ruff format .
```

## Architecture

**Plan-and-Execute** agent implemented with LangGraph, targeting a helpdesk use case over a fictional "XYZ system."

### Main Graph (`src/agent.py` - `HelpDeskAgent`)

```
create_plan --> [parallel via Send] execute_subtasks --> create_answer
```

- `create_plan`: Uses OpenAI Structured Output (`Plan` model) to decompose the question into subtasks.
- `execute_subtasks`: Spawns parallel subgraph executions (one per subtask) using LangGraph's `Send`.
- `create_answer`: Synthesizes all subtask answers into a final response.

### Subgraph (per subtask, retries up to `MAX_CHALLENGE_COUNT = 3`)

```
select_tools --> execute_tools --> create_subtask_answer --> reflect_subtask
                     ^                                              |
                     |____________ (retry if not complete) _________|
```

`reflect_subtask` uses Structured Output (`ReflectionResult`) to judge if the answer is adequate. On failure it feeds advice back into `select_tools`.

### Tools (`src/tools/`)

- `search_xyz_manual`: Full-text keyword search via Elasticsearch (port 9200), index `documents`. Uses kuromoji analyzer for Japanese.
- `search_xyz_qa`: Semantic vector search via Qdrant (port 6333), collection `documents`, using `text-embedding-3-small` embeddings.

Both return `list[SearchOutput]` (fields: `file_name`, `content`).

### Index creation (`src/scripts/create_index.py`)

- Loads PDFs from `data/` into Elasticsearch (chunked, keyword search).
- Loads CSV QA pairs from `data/` into Qdrant (vector embeddings, semantic search).

### Key Files

| File | Purpose |
|------|---------|
| `src/agent.py` | `HelpDeskAgent` class: graph/subgraph definitions and all node methods |
| `src/models.py` | Pydantic models: `Plan`, `Subtask`, `ReflectionResult`, `AgentResult`, `SearchOutput`, `ToolResult` |
| `src/prompts.py` | All prompt templates in `HelpDeskAgentPrompts`; prompts are in Japanese |
| `src/configs.py` | `Settings` (reads `.env`) |
| `notebooks/entire_graph_runner.ipynb` | End-to-end agent execution |
| `notebooks/flow_steps_runner.ipynb` | Step-by-step graph execution |
| `notebooks/tools.ipynb` | Tool testing |

### State Types

- `AgentState`: top-level graph state (`question`, `plan`, `subtask_results`, `last_answer`)
- `AgentSubGraphState`: per-subtask state (`messages`, `tool_results`, `reflection_results`, `challenge_count`, `is_completed`)

### Infrastructure

Docker Compose (`docker-compose.yml`) runs Elasticsearch and Qdrant. ES data is persisted to `.rag_data/es_data/`. If volume mount errors occur on Docker, comment out the volume in `docker-compose.yml`.
