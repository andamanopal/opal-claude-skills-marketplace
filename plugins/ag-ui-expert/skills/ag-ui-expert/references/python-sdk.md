# Python SDK Reference

## Installation

```bash
pip install ag-ui-protocol
```

## Core Imports

```python
from ag_ui.core import (
    # Input/Output
    RunAgentInput,
    
    # Event Types Enum
    EventType,
    
    # Lifecycle Events
    RunStartedEvent,
    RunFinishedEvent,
    RunErrorEvent,
    StepStartedEvent,
    StepFinishedEvent,
    
    # Text Events
    TextMessageStartEvent,
    TextMessageContentEvent,
    TextMessageEndEvent,
    
    # Tool Events
    ToolCallStartEvent,
    ToolCallArgsEvent,
    ToolCallEndEvent,
    ToolCallResultEvent,
    
    # State Events
    StateSnapshotEvent,
    StateDeltaEvent,
    MessagesSnapshotEvent,
    
    # Custom Events
    CustomEvent,
    RawEvent,
    
    # Message Types
    Message,
    UserMessage,
    AssistantMessage,
    SystemMessage,
    ToolMessage,
)
from ag_ui.encoder import EventEncoder
```

## EventEncoder

Handles SSE formatting based on Accept header:

```python
encoder = EventEncoder()

# Encode single event → "data: {...json...}\n\n"
sse_data = encoder.encode(event)

# Usage in StreamingResponse
async def stream():
    yield encoder.encode(RunStartedEvent(...))
    yield encoder.encode(TextMessageContentEvent(...))
```

## Event Models

### Lifecycle Events

```python
# Run started - MUST be first event
RunStartedEvent(
    type=EventType.RUN_STARTED,
    thread_id="thread-123",
    run_id="run-456"
)

# Run finished - MUST be last event (success)
RunFinishedEvent(
    type=EventType.RUN_FINISHED,
    thread_id="thread-123",
    run_id="run-456"
)

# Run error - Terminal event (failure)
RunErrorEvent(
    type=EventType.RUN_ERROR,
    message="Something went wrong",
    code="INTERNAL_ERROR"  # INTERNAL_ERROR, VALIDATION_ERROR, AUTH_ERROR, RATE_LIMITED, TOOL_ERROR, TIMEOUT, CANCELLED
)

# Step markers for multi-step workflows
StepStartedEvent(
    type=EventType.STEP_STARTED,
    step_name="search_documents"
)
StepFinishedEvent(
    type=EventType.STEP_FINISHED,
    step_name="search_documents"
)
```

### Text Message Events

```python
# Start message - opens message context
TextMessageStartEvent(
    type=EventType.TEXT_MESSAGE_START,
    message_id="msg-789",
    role="assistant"  # "assistant" or "user"
)

# Content chunks - stream text incrementally
TextMessageContentEvent(
    type=EventType.TEXT_MESSAGE_CONTENT,
    message_id="msg-789",
    delta="Hello "  # Incremental text chunk
)

# End message - closes message context
TextMessageEndEvent(
    type=EventType.TEXT_MESSAGE_END,
    message_id="msg-789"
)
```

### Tool Call Events

```python
# Start tool call
ToolCallStartEvent(
    type=EventType.TOOL_CALL_START,
    tool_call_id="tool-001",
    tool_call_name="search_lifelogs",
    parent_message_id="msg-789"  # Optional: link to message
)

# Stream tool arguments (for large args)
ToolCallArgsEvent(
    type=EventType.TOOL_CALL_ARGS,
    tool_call_id="tool-001",
    delta='{"query": "meeting"}'  # JSON string chunk
)

# End tool call
ToolCallEndEvent(
    type=EventType.TOOL_CALL_END,
    tool_call_id="tool-001"
)

# Tool result
ToolCallResultEvent(
    type=EventType.TOOL_CALL_RESULT,
    tool_call_id="tool-001",
    result='{"entries": [...]}'  # JSON string
)
```

### State Events

```python
# Full state snapshot (initial sync)
StateSnapshotEvent(
    type=EventType.STATE_SNAPSHOT,
    snapshot={
        "results": [],
        "loading": True,
        "query": ""
    }
)

# Incremental state update (JSON Patch RFC 6902)
StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[
        {"op": "replace", "path": "/loading", "value": False},
        {"op": "add", "path": "/results/-", "value": {"id": 1, "text": "..."}}
    ]
)

# Messages snapshot (sync conversation history)
MessagesSnapshotEvent(
    type=EventType.MESSAGES_SNAPSHOT,
    messages=[
        {"role": "user", "content": "Hello"},
        {"role": "assistant", "content": "Hi there!"}
    ]
)
```

### Custom Events

```python
# Custom named event
CustomEvent(
    type=EventType.CUSTOM,
    name="INTERRUPT_APPROVAL_REQUIRED",
    value={
        "action": "delete_entry",
        "options": ["approve", "reject"]
    }
)

# Raw passthrough
RawEvent(
    type=EventType.RAW,
    event={"custom": "data"}
)
```

## Message Types

Six message roles available:

```python
# User message
UserMessage(role="user", content="Search for meeting notes")

# Assistant message
AssistantMessage(role="assistant", content="I found 5 entries...")

# System message
SystemMessage(role="system", content="You are a helpful assistant")

# Developer message (internal instructions)
DeveloperMessage(role="developer", content="Use formal tone")

# Tool message (tool results)
ToolMessage(
    role="tool",
    tool_call_id="tool-001",
    content='{"results": [...]}'
)

# Activity message (status updates)
ActivityMessage(role="activity", content="Searching...")
```

## RunAgentInput Fields

```python
class RunAgentInput(BaseModel):
    thread_id: str                      # Conversation ID
    run_id: str                         # Unique run ID
    parent_run_id: Optional[str]        # For nested runs
    state: Any                          # Frontend state object
    messages: List[Message]             # Conversation history
    tools: List[Tool]                   # Frontend-provided tools
    context: List[Context]              # Additional context
    forwarded_props: Any                # Custom properties
```

### Accessing Input Data

```python
@app.post("/agent")
async def agent(input_data: RunAgentInput):
    # Get conversation history
    for msg in input_data.messages:
        print(f"{msg.role}: {msg.content}")
    
    # Get frontend state
    current_state = input_data.state
    
    # Get auth from forwarded props
    auth_token = input_data.forwarded_props.get("auth_token")
    
    # Get available tools
    for tool in input_data.tools:
        print(f"Tool: {tool.name} - {tool.description}")
```

## Complete Streaming Example

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from ag_ui.core import *
from ag_ui.encoder import EventEncoder
import uuid
import asyncio

app = FastAPI()

@app.post("/agent")
async def agent(input_data: RunAgentInput):
    encoder = EventEncoder()
    
    async def stream():
        # 1. Start run
        yield encoder.encode(RunStartedEvent(
            type=EventType.RUN_STARTED,
            thread_id=input_data.thread_id,
            run_id=input_data.run_id
        ))
        
        # 2. Initial state
        yield encoder.encode(StateSnapshotEvent(
            type=EventType.STATE_SNAPSHOT,
            snapshot={"loading": True, "results": []}
        ))
        
        # 3. Start message
        msg_id = str(uuid.uuid4())
        yield encoder.encode(TextMessageStartEvent(
            type=EventType.TEXT_MESSAGE_START,
            message_id=msg_id,
            role="assistant"
        ))
        
        # 4. Stream content
        response = "Here are your results..."
        for word in response.split():
            yield encoder.encode(TextMessageContentEvent(
                type=EventType.TEXT_MESSAGE_CONTENT,
                message_id=msg_id,
                delta=word + " "
            ))
            await asyncio.sleep(0.05)  # Simulate streaming
        
        # 5. End message
        yield encoder.encode(TextMessageEndEvent(
            type=EventType.TEXT_MESSAGE_END,
            message_id=msg_id
        ))
        
        # 6. Final state
        yield encoder.encode(StateDeltaEvent(
            type=EventType.STATE_DELTA,
            delta=[{"op": "replace", "path": "/loading", "value": False}]
        ))
        
        # 7. End run
        yield encoder.encode(RunFinishedEvent(
            type=EventType.RUN_FINISHED,
            thread_id=input_data.thread_id,
            run_id=input_data.run_id
        ))
    
    return StreamingResponse(stream(), media_type="text/event-stream")
```
