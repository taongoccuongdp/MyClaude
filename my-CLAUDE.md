# Global Python Coding Standards

## Working Philosophy

1. **Focus on code and planning** - Spend 80% of effort on implementation and planning, not documentation
2. **Ship fast** - Prioritize implementing key features over handling every edge case
3. **Keep it simple** - Write concise, easy-to-understand code
4. **Simple > optimal** - Choose clear, straightforward solutions over complex optimal ones

**Communication**: ALWAYS ask the user questions to clarify anything that is unclear or when you need them to provide additional information.

## Python Essentials

### Pydantic
```python
# ✅ CORRECT
model.model_dump()      # NOT .dict()
model.model_copy()      # NOT .copy()

class Output(BaseModel):
    """Structured LLM output."""
    field: str = Field(description="Natural language desc")

llm.with_structured_output(Output, method="function_calling")
```

### Strings & Logging
```python
# ✅ CORRECT - Loguru only (NOT Python logging)
from loguru import logger
logger.info("Processing {}", job_id)    # Placeholders, not f-strings
my_string = "Static string"              # No f-prefix without placeholders

# ❌ WRONG
logger.info(f"Processing {job_id}")
my_string = f"Static string"

# Standard loguru setup
logger.add(sys.stdout, level="INFO",
           format="<green>{time}</green> <level>{message}</level>",
           serialize=False, enqueue=True, backtrace=True, diagnose=True)
```

## LangGraph Framework

**Docs**: Use `langgraph-docs` MCP server for ALL LangGraph questions:
1. `list_doc_sources` → get llms.txt
2. `fetch_docs` → read relevant pages

**Core Principles**:
- Fixed workflows (state machines) > flexible agents (predictability)
- Explicit state (Pydantic models) > implicit (messages)
- All-at-once field collection > one-by-one (better UX)
- Batch validation (show ALL errors together)
- Hybrid intelligence: LLM for parsing/generation, fixed logic for control

**Patterns**:
```python
from langgraph.prebuilt import ToolNode
from langgraph.checkpoint.postgres import PostgresCheckpointer

# Always use ToolNode for tools
graph.add_node("tools", ToolNode(tools))

# PostgreSQL checkpointer (production)
checkpointer = PostgresCheckpointer(connection_string=f"postgresql://{host}/{db}")
graph = graph.compile(checkpointer=checkpointer)

# Handle interrupts at application level (NEVER try-catch)
try:
    result = graph.invoke({"message": user_input}, ...)
except GraphInterrupt as e:
    user_response = get_user_input(e.value)
    result = graph.invoke({"message": user_response},
                         config={"checkpoint_id": e.checkpoint_id})
```

**Workflow Best Practices**:
- ✅ DO: All-at-once field collection, batch validation, factory functions, explicit state
- ❌ DON'T: One-by-one fields, first-error-only, duplicate logic, implicit state, catch GraphInterrupt

## Syntax Rules

### Prompts & JSON
```python
# ✅ CORRECT
prompt = "Return JSON: {{'status': '...', 'result': '...'}}"  # {{ }} for JSON objects
prompt = "Process: {user_input}"                              # { } for placeholders

# ❌ WRONG
prompt = "Return JSON: {'status': '...', 'result': '...'}"
```

### Type Checking
```python
# ✅ CORRECT - Quote types from TYPE_CHECKING
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from module import HeavyType

async def process(value: "HeavyType") -> None:  # Quoted!
    pass

# ❌ WRONG
async def process(value: HeavyType) -> None:  # NameError at runtime
    pass
```

### Dependency Injection
```python
# Constructor signatures vary - ALWAYS verify!
UserRepository(db_config, unit)     # Takes unit param
ChatInfoRepo(db_config)             # NO unit param

# DI: Match param names to registered instances
# Register by both name and class name for flexibility
```

## Quick Reference

**Common Gotchas**:
1. Quote TYPE_CHECKING types (`"FormatType"`)
2. Use `model_dump()` not `.dict()`
3. Loguru placeholders `{}` not f-strings
4. No f-prefix without placeholders
5. NEVER catch GraphInterrupt
6. Verify constructor signatures (unit param varies)
