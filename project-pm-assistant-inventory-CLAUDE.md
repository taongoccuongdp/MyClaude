# PM Assistant Inventory - Development Guide

Python 3.13.6 | PostgreSQL | Google Sheets | LangGraph | Lotus Chat Bot (Telegram-based)

**Key**: Inventory branch uses PostgreSQL (main project uses MySQL). Always work from `/inventory` directory.

## Quick Start

```bash
# Always set PYTHONPATH=.
PYTHONPATH=. python tests/test_<feature>.py
PYTHONPATH=. python main/job_scheduler_main.py    # Job scheduler
PYTHONPATH=. python main/lotus_chat_bot.py        # Chatbot
```

## Architecture Layers

**1. Entities** (`src/app/entities/`): Domain models (JobPlan, User, Team, TeamPlan, GSheetTable)

**2. Repositories** (`src/app/repositories/`): Data access with interface pattern
- PostgreSQL: `postgres/` - Main database (users, teams, scheduled_jobs, chat data)
- Google Sheets: `google_sheet/` - Job data source
- Pattern: `async with await self.get_connection() as conn: ...`

**3. Use Cases** (`src/app/use_cases/`, `src/app/use_cases_v2/`): Business logic
- `use_cases/lotus_chat_handler/`: Real-time chat processing
- `use_cases_v2/plannings/`: Monthly plan creation
- `use_cases_v2/job_monitors/`: Job monitoring & reporting
- `use_cases_v2/job_schedulers/`: Scheduling system

**4. LangGraph Agents** (`src/app/agents/`): AI workflows
- `chatbot/workflows/`: Fixed workflows (Agentic Process Automation)
- `tools/`: LangChain tools (repository_tools, scheduler_tools)

**5. Services** (`src/google_services/`, `src/lotus_chat_bot.py`): External integrations

## Critical Conventions

### Code Style
```python
# ✅ CORRECT
from loguru import logger
logger.info("Processing {}", job_id)           # Placeholders, not f-strings
my_string = "Static string"                     # No f-prefix without placeholders
model.model_dump()                              # NOT .dict()
model.model_copy()                              # NOT .copy()

# ❌ WRONG
logger.info(f"Processing {job_id}")
my_string = f"Static string"
model.dict()
model.copy()
```

### Pydantic Models
```python
from pydantic import BaseModel, Field

class JobSummary(BaseModel):
    """Job summary structured output."""
    team_name: str = Field(description="Team name")
    total_jobs: int = Field(description="Total jobs")

llm.with_structured_output(JobSummary, method="function_calling")
```

### LangGraph Patterns
```python
from langgraph.prebuilt import ToolNode
from langgraph.graph import StateGraph

# Fixed workflows (preferred): init → collect → validate → execute
# State: Explicit Pydantic models (not implicit in messages)
# Collection: All-at-once field parsing (better UX)
# Validation: Show ALL errors together
# Human-in-loop: Use interrupt/Command (NEVER catch GraphInterrupt)

graph.add_node("tools", ToolNode(tools))  # Always use ToolNode
```

**LangGraph Docs**: Use `langgraph-docs` MCP server for ANY LangGraph questions:
1. `list_doc_sources` → get llms.txt
2. `fetch_docs` → read index & relevant pages

### Naming Conventions
```python
# Repository Classes (INCONSISTENT - always verify!)
from src.app.repositories.postgres.user_repo import UserRepository
from src.app.repositories.postgres.chat_info_repo import ChatInfoRepo  # NOT Repository

# Services
from src.google_services import GoogleSpreadSheetsService  # NOT GoogleSheetsService
from src.lotus_chat_bot import LotusChatBase              # NOT LotusChatBot

# Constructor signatures VARY - verify before DI!
UserRepository(db_config, unit)     # Takes unit
ChatInfoRepo(db_config)             # NO unit parameter
```

### Polars DataFrames
```python
df.fill_null("")
df.group_by("team").agg([pl.when(...).then(1).otherwise(0).sum()])
df.sort(["deadline"], nulls_last=[True])
df.drop("serial").with_row_index("serial", offset=1)
```

### Prompts
- JSON objects: `{{ }}` (double braces)
- Placeholders: `{ }` (single braces)

## Configuration System

**File**: `settings.yaml` → Pydantic models in `src/app/settings.py`

```python
from src.app.settings import settings

# Access typed config
db_config = settings.pm_assistant_postgresql_db
bot_id = settings.job_assistant_bot.bot_id
unit = settings.unit  # "admicro.inventory"

# Initialize repositories
user_repo = UserRepository(settings.pm_assistant_postgresql_db, settings.unit)
```

**Key sections**: `pm_assistant_postgresql_db`, `task_postgres_db`, `plan_sheet_template`, `details_sheet_template`, `job_assistant_bot`, `lotus_chat_handler`, `openai_api_key`, `unit`

## Job Scheduler (Dual-Mode)

**Mode 1: Simple Functions** (backward compatible)
```python
async def my_job(param1: str) -> dict:
    return {"status": "completed"}

runner.register_function("my_job", my_job)
```

**Mode 2: Usecase Classes** (user-configurable via `/scheduler`)
```python
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from src.lotus_chat_bot import FormatType  # Avoid NameError

class MyUseCase(Usecase):
    """Brief description."""

    ALLOW_CONFIG_BY_USER = True  # Enable chatbot scheduling

    def __init__(self, user_repo, sheet_service):  # DI by param name
        self._user_repo = user_repo
        self._sheet_service = sheet_service

    async def __call__(
        self,
        team_name: str,
        output_chats: list[str],
        parse_mode: "FormatType" = "TEXT"  # QUOTE types from TYPE_CHECKING
    ) -> dict:
        """
        Use case description.

        Args:
            team_name: Natural language description (shown to users)
            output_chats: List of chat titles to send to (special handling)
            parse_mode: Message format (HTML/TEXT)
        """
        # Implementation
        return {"status": "completed"}
```

**Job Runner**: Detects `usecase:` prefix → Dynamic import → DI container → Execute

**DI Container** (`src/app/use_cases_v2/_usecase_di.py`):
- Singleton pattern
- Matches `__init__` param names to registered instances
- Verify constructor signatures (some take `unit`, some don't)

**Database**: `scheduled_jobs` table, function_name = `"usecase:full.module.path.ClassName"`

**JSON Parsing**: Auto-handles single/double-escaped JSONB values

## Chatbot Integration

**File**: `main/lotus_chat_bot.py`
- Polling: 2s via GetUpdates
- Handler: `AgentChatHandler` (Celery async processing)
- Checkpointer: PostgreSQL conversation state
- Format: HTML with `parse_mode="HTML"`

**Fixed Workflows** (`src/app/agents/chatbot/workflows/`):
```
START → init → collect_fields → validate → execute → success
              ↑                           ↓
              └───── error_recovery ←─────┘
```

**Components**:
- `base.py`: Factory functions (create_parse_node, create_validation_node, etc.)
- `models.py`: Pydantic state models (is_complete(), missing_fields())
- `router.py`: Pattern matching + LLM fallback

## Data Flow: Google Sheets → Polars → Processing → Write Back

```python
# Read with GoogleSpreadSheetsService (render_option="FORMATTED_VALUE")
job_raw_data = sheet_service.get_values(spreadsheet_id, sheet_name, range)

# Parse to Polars DataFrame (21 columns for JobPlan)
jobs = job_details_repo.get_job_details_from_sheet(job_raw_data, team_id)

# Transform & write back
```

**Done Statuses**: `["Beta", "Close", "Done"]`

## Development Patterns

### Adding Fixed Workflow
1. Define Pydantic model in `workflows/models.py` (is_complete, missing_fields)
2. Create workflow in `workflows/my_workflow.py` (use factory functions)
3. Export in `workflows/__init__.py`
4. Register in `graph.py`
5. Add route in `router.py` COMMAND_MAP

### Adding Schedulable Use Case
1. Create class in `src/app/use_cases_v2/{domain}/`
2. Set `ALLOW_CONFIG_BY_USER = True`
3. Write comprehensive Args: docstring (shown to users)
4. Quote TYPE_CHECKING type annotations
5. Test: `discover_schedulable_use_cases()`

### JobHandler Pattern
```python
class MyMonitor(JobHandler):
    async def handle_teamplan(self, teamplan, jobs, team, leader, ...):
        # Custom logic - base class handles data loading & filtering
        pass
```

## Database Schema

**PostgreSQL Tables**:
- `users`, `teams`, `team_plans`, `manager_groups`
- `scheduled_jobs` (new scheduler system)
- `chat_info`, `lotus_chat_messages`
- Task tables in `task_postgres_db`

**Multi-tenancy**: All queries filter by `unit = "admicro.inventory"`

## Common Pitfalls

1. **FormatType NameError**: Quote types from TYPE_CHECKING (`"FormatType"`)
2. **DI Injection Fails**: Verify constructor signatures (unit param varies)
3. **Double-escaped JSON**: Repository handles automatically
4. **User sees variable names**: Write natural Args: docstrings
5. **GraphInterrupt caught**: NEVER try-catch interrupt (must propagate)
6. **f-strings in logger**: Use placeholders `logger.info("{}", value)`
7. **Wrong class names**: Verify imports (ChatInfoRepo not ChatInfoRepository)

## Troubleshooting

```bash
# Verify class names
grep "^class " src/app/repositories/postgres/chat_info_repo.py

# Check constructor signature
grep -A 5 "def __init__" src/app/repositories/postgres/user_repo.py

# Test use case discovery
PYTHONPATH=. python -c "from src.app.use_cases_v2._introspection import discover_schedulable_use_cases; print([uc.class_name for uc in discover_schedulable_use_cases()])"

# Test DI container
from src.app.use_cases_v2._usecase_di import UsecaseDIContainer
container = UsecaseDIContainer()
print(container._instances.keys())
```

## Key Files Reference

- Config: `src/app/settings.py`, `settings.yaml`
- Scheduler: `main/job_scheduler_main.py`, `src/job_scheduler/runner.py`
- Chatbot: `main/lotus_chat_bot.py`, `src/app/agents/chatbot/`
- DI: `src/app/use_cases_v2/_usecase_di.py`
- Introspection: `src/app/use_cases_v2/_introspection.py`
- Prompts: `src/prompts/`

---

**Project Principles**:
- Fixed workflows > flexible agents (predictability)
- All-at-once collection > one-by-one (UX)
- Explicit state (Pydantic) > implicit (messages)
- Batch validation > first-error-only
- Natural language > technical jargon (user-facing)
