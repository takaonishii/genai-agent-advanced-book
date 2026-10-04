# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Structure

This repository contains code for the book "現場で活用するためのAIエージェント実践入門" (Practical Introduction to AI Agents). Currently only `chapter4/` is present.

## Chapter 4 Setup

All commands below should be run from `chapter4/`.

### Dependencies

```bash
cd chapter4
uv sync                        # Install dependencies
source .venv/bin/activate      # Activate virtual environment
```

### Environment Variables

Create `chapter4/.env`:
```env
OPENAI_API_KEY=your_openai_api_key
OPENAI_API_BASE="https://api.openai.com/v1"
OPENAI_MODEL="gpt-4o-2024-08-06"
```

`configs.py` reads the `.env` from path `./chapter4/.env` via `pydantic-settings`.

### Search Index Setup (required before running agent)

```bash
make start.engine    # Start Elasticsearch + Qdrant via Docker Compose
make create.index    # Build search indexes (runs src/scripts/create_index.py)
make delete.index    # Delete indexes
make stop.engine     # Stop containers
```

### Linting

```bash
uv run ruff check .       # Lint
uv run ruff format .      # Format
```

## Architecture

The agent is a **Plan-and-Execute** pattern implemented with LangGraph, targeting a helpdesk use case over a fictional "XYZ system."

### Main Graph (`src/agent.py` - `HelpDeskAgent`)

```
create_plan --> [parallel] execute_subtasks --> create_answer
```

- `create_plan`: Uses OpenAI Structured Output (`Plan` model) to decompose the user's question into subtasks.
- `execute_subtasks`: Spawns parallel subgraph executions (one per subtask) using LangGraph's `Send`.
- `create_answer`: Synthesizes all subtask answers into a final response.

### Subgraph (per subtask)

```
select_tools --> execute_tools --> create_subtask_answer --> reflect_subtask
                     ^                                              |
                     |____________ (retry if not complete) _________|
```

- Retries up to `MAX_CHALLENGE_COUNT = 3` times.
- `reflect_subtask` uses Structured Output (`ReflectionResult`) to judge if the subtask answer is adequate. If not, it re-selects tools with feedback.

### Tools (`src/tools/`)

- `search_xyz_manual`: Full-text keyword search via **Elasticsearch** (port 9200), index `documents`.
- `search_xyz_qa`: Semantic vector search via **Qdrant** (port 6333), collection `documents`, using `text-embedding-3-small` embeddings.

Both tools are LangChain `@tool`-decorated functions returning `list[SearchOutput]`.

### Key Files

| File | Purpose |
|------|---------|
| `src/agent.py` | `HelpDeskAgent` class with graph/subgraph definitions and node methods |
| `src/models.py` | Pydantic models: `Plan`, `Subtask`, `ReflectionResult`, `AgentResult`, `SearchOutput` |
| `src/prompts.py` | All prompt templates (`HelpDeskAgentPrompts`) |
| `src/configs.py` | `Settings` (pydantic-settings, reads from `.env`) |
| `notebooks/entire_graph_runner.ipynb` | End-to-end agent execution notebook |
| `notebooks/flow_steps_runner.ipynb` | Step-by-step graph execution notebook |
| `notebooks/tools.ipynb` | Tool testing notebook |

### Infrastructure

Docker Compose (`docker-compose.yml`) runs Elasticsearch and Qdrant. Elasticsearch data is persisted to `.rag_data/es_data/`. If volume mount errors occur, comment out the volume in `docker-compose.yml` (data won't persist across container restarts).
