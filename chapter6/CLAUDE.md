# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

All commands should be run from `chapter6/`.

```bash
uv sync                        # Install dependencies
cp .env.sample .env            # Create env file and fill in keys
```

### Required environment variables (`.env`)

```
OPENAI_API_KEY=...
ANTHROPIC_API_KEY=...
COHERE_API_KEY=...
JINA_API_KEY=...
```

Optional: `LANGSMITH_API_KEY`, `LANGSMITH_TRACING_V2=true`

## Running

```bash
# Launch LangGraph Studio (http://localhost:8123)
uv run langgraph dev --no-reload

# Test individual subgraphs from CLI
uv run python -m arxiv_researcher.agent.paper_analyzer_agent fixtures/2408.14317.md
uv run python -m arxiv_researcher.agent.paper_search_agent
```

## Architecture

This is a **multi-agent arXiv researcher** built with LangGraph. Given a research goal, it interactively clarifies the goal, searches arXiv, reads papers, and generates a structured report.

### Agent Hierarchy

```
ResearchAgent (main graph, with MemorySaver checkpointer)
  user_hearing → human_feedback (interrupt) → goal_setting
    → decompose_query → paper_search_agent → evaluate_task
    → [retry loop if insufficient] → generate_report
      │
      └─► PaperSearchAgent (subgraph)
              search_papers (PaperProcessor, parallel fetch + PDF→MD)
                → analyze_paper → organize_results
                      │
                      └─► PaperAnalyzerAgent (subgraph per paper)
                              set_section → check_sufficiency
                              → [retry up to 3x] → summarize / mark_as_not_related
```

### Key design decisions

- **Three LLMs**: `settings.llm` (gpt-4o, complex tasks), `settings.fast_llm` (gpt-4o-mini, search/analysis), `settings.reporter_llm` (claude-3-7-sonnet, final report generation)
- **Cohere reranking**: After arXiv search, papers are reranked by Cohere and filtered by relevance score ≥ 0.7
- **PDF→Markdown**: Jina Reader API converts PDFs to Markdown, stored in `storage/markdown/`
- **Human-in-the-loop**: `ResearchAgent` uses LangGraph `interrupt` at `human_feedback` node to clarify research goals before proceeding
- **Task retry loop**: `evaluate_task` checks if gathered papers are sufficient; retries `paper_search_agent` up to `max_evaluation_retry_count=3` times

### Key files

| File | Purpose |
|------|---------|
| `arxiv_researcher/agent/research_agent.py` | Main graph (`ResearchAgent`); `graph` exported for LangGraph Studio |
| `arxiv_researcher/agent/paper_search_agent.py` | Subgraph that searches arXiv and dispatches to analyzer |
| `arxiv_researcher/agent/paper_analyzer_agent.py` | Subgraph that reads a single paper and decides relevance |
| `arxiv_researcher/chains/reading_chains.py` | `SetSection`, `CheckSufficiency`, `Summarizer` nodes |
| `arxiv_researcher/searcher/arxiv_searcher.py` | arXiv API search with query expansion, date filtering, Cohere reranking |
| `arxiv_researcher/service/pdf_to_markdown.py` | Jina Reader integration for PDF conversion |
| `arxiv_researcher/settings.py` | `Settings` (pydantic-settings); all config and LLM instances |
| `arxiv_researcher/chains/prompts/` | All prompt templates as `.prompt` files |
| `langgraph.json` | Registers all three graphs for LangGraph Studio |
