# Agentic AI System — Backend

Production-grade single-agent system with **ReAct loop**, **tool calling**, **SSE streaming**, and **persistent threads**.

Built with **LangChain + FastAPI**, supporting multiple LLM providers (Anthropic, OpenAI, Google, Ollama, etc.), following **SOLID principles** and designed for future multi-agent extension.

## Architecture

```
User Query → Thread Store → Context Assembly → ReAct Loop → SSE Stream → UI
```

- **Thread Store**: Persistent conversation threads + message history (PostgreSQL)
- **Context Assembly**: System prompt + rules + tool specs + full chat history
- **ReAct Loop**: Think → Decide → Act → Observe → Repeat
- **SSE Stream**: Typed events (thinking, text, tool calls) pushed in real time

## Project Structure

```
app/
├── config/
│   ├── settings.py           # Environment config (Pydantic Settings)
│   ├── system_prompt.txt     # Agent identity + behavior
│   └── rules.txt             # Hard constraints
├── agent/
│   ├── runner.py             # AgentRunner — ReAct loop orchestrator
│   ├── context_assembler.py  # Builds full context per turn
│   └── prompt_builder.py     # Formats context → LangChain messages
├── tools/
│   ├── registry.py           # ToolRegistry — injectable, not a singleton
│   ├── calculator.py         # Math expression evaluator
│   └── file_reader.py        # Local file reader
├── streaming/
│   ├── sse_handler.py        # Formats events → SSE wire format
│   └── event_mapper.py       # Maps internal events → SSE event names
├── threads/
│   ├── thread_store.py       # PostgreSQL-backed thread + message persistence
│   └── models.py             # Thread, Message ORM models
├── core/
│   └── db/
│       └── session.py        # Async SQLAlchemy session factory
├── api/
│   ├── stream_routes.py      # Thread CRUD + SSE streaming endpoints
│   ├── health.py             # /health liveness/readiness endpoints
│   └── router.py             # Central API router
├── bootstrap.py              # Component initialization + DI wiring
└── main.py                   # FastAPI app entrypoint
```

## Setup

### 1. Install Dependencies

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS/Linux

pip install -r requirements.txt
```

### 2. Configure Environment

```bash
copy .env.example .env
# Edit .env — set your API_KEY and DATABASE_URL
```

**Required env vars:**

| Variable | Description | Default |
|----------|-------------|---------|
| `API_KEY` | API key for your chosen LLM provider | Required |
| `MODEL_PROVIDER` | LLM provider (`anthropic`, `openai`, `google_genai`, `ollama`, …) | `anthropic` |
| `MODEL_NAME` | Model name to use | `claude-sonnet-4-6` |
| `DATABASE_URL` | PostgreSQL connection string | `postgresql+asyncpg://postgres:postgres@localhost:5432/agentic_ai` |
| `MAX_TOKENS` | Max tokens per response | `4096` |
| `MAX_ITERATIONS` | ReAct loop iteration limit | `10` |
| `MAX_EXECUTION_TIME` | Max allowed time per run (seconds) | `60` |
| `DEBUG` | Enable debug mode + auto-reload | `false` |

### 3. Run Database Migrations

```bash
alembic upgrade head
```

### 4. Run

```bash
uvicorn app.main:app --reload
```

Visit: http://localhost:8000/api/docs

### 5. Use the API

**Create a thread first**, then stream into it:

```bash
# 1. Create a thread
curl -X POST http://localhost:8000/api/v1/threads \
  -H "Content-Type: application/json" \
  -d '{"title": "My conversation"}'

# 2. Stream a query into that thread
curl "http://localhost:8000/api/v1/threads/{thread_id}/stream?query=What+is+2+plus+2"
```

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/threads` | Create a new conversation thread |
| `GET` | `/api/v1/threads` | List all threads |
| `GET` | `/api/v1/threads/{thread_id}` | Get thread details |
| `DELETE` | `/api/v1/threads/{thread_id}` | Delete a thread and all its messages |
| `GET` | `/api/v1/threads/{thread_id}/messages` | Get all messages in a thread |
| `GET` | `/api/v1/threads/{thread_id}/stream` | SSE — stream an agent run |
| `GET` | `/health` | Basic health check |
| `GET` | `/health/liveness` | Liveness probe |
| `GET` | `/health/readiness` | Readiness probe (checks DB) |

## SSE Event Types

| SSE Event Name | Description |
|----------------|-------------|
| `thread_info` | Thread ID for the conversation |
| `content_block_start` | LLM begins a new block (text/thinking) |
| `content_block_delta` | Streaming token (`text_delta`, `thinking_delta`, `input_json_delta`) |
| `content_block_stop` | Block complete |
| `tool_execution` | Tool is being called (name + input) |
| `tool_result` | Tool returned a result |
| `message_delta` | Turn complete (`end_turn` / `max_iterations`) |
| `message_stop` | Stream ends — close connection |

All events carry a `source_agent` field (`"primary"` in v1).

## How to Add a New Tool

1. Create `app/tools/your_tool.py`:

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field

class YourToolInput(BaseModel):
    param: str = Field(description="What this param does")

class YourTool(BaseTool):
    name: str = "your_tool"
    description: str = "What it does + when to use it"
    args_schema: type[BaseModel] = YourToolInput

    def _run(self, param: str) -> str:
        return "result"

    async def _arun(self, param: str) -> str:
        return self._run(param)
```

2. Register in `app/bootstrap.py`:

```python
from app.tools.your_tool import YourTool
registry.register(YourTool())
```

3. Done. Tool spec is auto-injected into the system prompt.

## Testing

```bash
pytest
```

## Multi-Agent Readiness

The architecture is designed for future multi-agent orchestration without requiring changes to existing code. The groundwork is in place but not yet wired up:

- **`AgentRunner` is stateless + injectable** — not a singleton; you can instantiate N of them independently
- **`ToolRegistry` is injectable, not a singleton** — each agent *can* receive its own registry with a different tool set, but currently a single shared registry is used
- **`ThreadStore` is thread-scoped** — conversation history is isolated per thread ID, ready to be sliced per agent
- **All SSE events carry `source_agent`** — currently always `"primary"`, but the field is in place for routing events from multiple agents
- **Future**: An `OrchestratorAgent` could instantiate N `AgentRunner` objects, each with its own `ToolRegistry`, and fan out/collect results

## License

MIT
