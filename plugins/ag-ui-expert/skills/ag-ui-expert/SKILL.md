---
name: ag-ui-expert
description: Expert-level AG-UI protocol implementation for building real-time agent-user interfaces. Use when implementing SSE-based agent streaming, LangGraph-to-frontend integration, CopilotKit setup, bidirectional state sync with JSON Patch, human-in-the-loop patterns, or any agent UI requiring real-time event streaming. Covers Python SDK, event types, middleware, security hardening, and production deployment.
---

# AG-UI Protocol Expert

AG-UI (Agent-User Interaction Protocol) is an event-sourcing protocol for real-time agent-user communication over SSE/WebSocket. Developed by CopilotKit in partnership with LangGraph and CrewAI.

## Protocol Position

```
MCP:   Agent ↔ Tools (databases, APIs)
A2A:   Agent ↔ Agent (multi-agent delegation)  
AG-UI: Agent ↔ User (real-time UI sync - "last mile")
```

## Quick Start - Minimal Endpoint

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from ag_ui.core import RunAgentInput, EventType
from ag_ui.core import RunStartedEvent, RunFinishedEvent
from ag_ui.core import TextMessageStartEvent, TextMessageContentEvent, TextMessageEndEvent
from ag_ui.encoder import EventEncoder
import uuid

app = FastAPI()

@app.post("/agent")
async def agent(input_data: RunAgentInput):
    encoder = EventEncoder()
    
    async def stream():
        yield encoder.encode(RunStartedEvent(
            type=EventType.RUN_STARTED,
            thread_id=input_data.thread_id,
            run_id=input_data.run_id
        ))
        
        msg_id = str(uuid.uuid4())
        yield encoder.encode(TextMessageStartEvent(
            type=EventType.TEXT_MESSAGE_START,
            message_id=msg_id,
            role="assistant"
        ))
        
        for chunk in process_llm_stream():  # Your LLM call
            yield encoder.encode(TextMessageContentEvent(
                type=EventType.TEXT_MESSAGE_CONTENT,
                message_id=msg_id,
                delta=chunk
            ))
        
        yield encoder.encode(TextMessageEndEvent(
            type=EventType.TEXT_MESSAGE_END,
            message_id=msg_id
        ))
        yield encoder.encode(RunFinishedEvent(
            type=EventType.RUN_FINISHED,
            thread_id=input_data.thread_id,
            run_id=input_data.run_id
        ))
    
    return StreamingResponse(stream(), media_type="text/event-stream")
```

## Event Types Quick Reference

| Category | Events | Purpose |
|----------|--------|---------|
| Lifecycle | `RUN_STARTED`, `RUN_FINISHED`, `RUN_ERROR` | Run boundaries |
| Steps | `STEP_STARTED`, `STEP_FINISHED` | Multi-step workflows |
| Text | `TEXT_MESSAGE_START`, `TEXT_MESSAGE_CONTENT`, `TEXT_MESSAGE_END` | Streaming text |
| Tools | `TOOL_CALL_START`, `TOOL_CALL_ARGS`, `TOOL_CALL_END`, `TOOL_CALL_RESULT` | Tool execution |
| State | `STATE_SNAPSHOT`, `STATE_DELTA`, `MESSAGES_SNAPSHOT` | Bidirectional sync |
| Custom | `CUSTOM`, `RAW` | Extensions |

**Minimal valid flow:**
```
RUN_STARTED → TEXT_MESSAGE_START → TEXT_MESSAGE_CONTENT* → TEXT_MESSAGE_END → RUN_FINISHED
```

## Core Input Schema

```python
class RunAgentInput(BaseModel):
    thread_id: str              # Conversation identifier
    run_id: str                 # Unique run identifier
    state: Any                  # Frontend state (bidirectional)
    messages: List[Message]     # Conversation history
    tools: List[Tool]           # Frontend-provided tools
    context: List[Context]      # Additional context
    forwarded_props: Any        # Custom properties (auth, config)
```

## Reference Documentation

For detailed implementation patterns, consult these references based on your needs:

- **Python SDK patterns**: See [references/python-sdk.md](references/python-sdk.md) for EventEncoder, event models, message types
- **LangGraph integration**: See [references/langgraph-integration.md](references/langgraph-integration.md) for event translation, async patterns
- **State management**: See [references/state-management.md](references/state-management.md) for JSON Patch (RFC 6902), bidirectional sync
- **Frontend integration**: See [references/frontend-integration.md](references/frontend-integration.md) for CopilotKit hooks, generative UI
- **Security & deployment**: See [references/security-deployment.md](references/security-deployment.md) for BFF pattern, nginx config, production hardening

## Common Patterns

### Error Handling

```python
async def stream():
    try:
        yield encoder.encode(RunStartedEvent(...))
        async for event in process_workflow(input_data):
            yield encoder.encode(event)
        yield encoder.encode(RunFinishedEvent(...))
    except asyncio.CancelledError:
        raise  # Client disconnected
    except Exception as e:
        yield encoder.encode(RunErrorEvent(
            type=EventType.RUN_ERROR,
            message=str(e),
            code="INTERNAL_ERROR"
        ))
```

### State Synchronization

```python
# Initial state snapshot
yield encoder.encode(StateSnapshotEvent(
    type=EventType.STATE_SNAPSHOT,
    snapshot={"results": [], "loading": True}
))

# Incremental updates via JSON Patch
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[{"op": "add", "path": "/results/-", "value": result}]
))
```

### Tool Calls

```python
tool_id = str(uuid.uuid4())
yield encoder.encode(ToolCallStartEvent(
    type=EventType.TOOL_CALL_START,
    tool_call_id=tool_id,
    tool_call_name="search_lifelogs"
))
yield encoder.encode(ToolCallArgsEvent(
    type=EventType.TOOL_CALL_ARGS,
    tool_call_id=tool_id,
    delta='{"query": "meeting notes"}'
))
result = await execute_tool("search_lifelogs", {"query": "meeting notes"})
yield encoder.encode(ToolCallEndEvent(
    type=EventType.TOOL_CALL_END,
    tool_call_id=tool_id
))
yield encoder.encode(ToolCallResultEvent(
    type=EventType.TOOL_CALL_RESULT,
    tool_call_id=tool_id,
    result=json.dumps(result)
))
```

### Human-in-the-Loop

```python
# Emit interrupt request
yield encoder.encode(CustomEvent(
    type=EventType.CUSTOM,
    name="INTERRUPT_APPROVAL_REQUIRED",
    value={
        "action": "delete_entry",
        "message": "Confirm deletion?",
        "options": ["approve", "reject"]
    }
))
# User response arrives in next RunAgentInput
```

## Installation

```bash
pip install ag-ui-protocol
```

## Resources

- Docs: https://docs.ag-ui.com
- GitHub: https://github.com/ag-ui-protocol/ag-ui
- AG-UI Dojo (testing): https://dojo.ag-ui.com
- CopilotKit: https://docs.copilotkit.ai
