# LangGraph Integration

## Overview

LangGraph emits its own event types that must be translated to AG-UI events. This reference covers the translation patterns and async workflow integration.

## Event Translation Map

| LangGraph Event | AG-UI Event(s) |
|-----------------|----------------|
| `on_chain_start` | `STEP_STARTED` |
| `on_chain_end` | `STEP_FINISHED` |
| `on_chat_model_start` | `TEXT_MESSAGE_START` |
| `on_chat_model_stream` | `TEXT_MESSAGE_CONTENT` |
| `on_chat_model_end` | `TEXT_MESSAGE_END` |
| `on_tool_start` | `TOOL_CALL_START` |
| `on_tool_end` | `TOOL_CALL_END`, `TOOL_CALL_RESULT` |

## Basic Translation Function

```python
from ag_ui.core import *
from ag_ui.encoder import EventEncoder
import uuid

async def translate_langgraph_event(
    lg_event: dict,
    encoder: EventEncoder,
    message_id: str
) -> AsyncGenerator[str, None]:
    """Translate single LangGraph event to AG-UI event(s)."""
    
    event_type = lg_event.get("event")
    data = lg_event.get("data", {})
    
    if event_type == "on_chain_start":
        step_name = lg_event.get("name", "workflow")
        yield encoder.encode(StepStartedEvent(
            type=EventType.STEP_STARTED,
            step_name=step_name
        ))
    
    elif event_type == "on_chain_end":
        step_name = lg_event.get("name", "workflow")
        yield encoder.encode(StepFinishedEvent(
            type=EventType.STEP_FINISHED,
            step_name=step_name
        ))
    
    elif event_type == "on_chat_model_start":
        yield encoder.encode(TextMessageStartEvent(
            type=EventType.TEXT_MESSAGE_START,
            message_id=message_id,
            role="assistant"
        ))
    
    elif event_type == "on_chat_model_stream":
        chunk = data.get("chunk")
        if chunk and hasattr(chunk, "content") and chunk.content:
            yield encoder.encode(TextMessageContentEvent(
                type=EventType.TEXT_MESSAGE_CONTENT,
                message_id=message_id,
                delta=chunk.content
            ))
    
    elif event_type == "on_chat_model_end":
        yield encoder.encode(TextMessageEndEvent(
            type=EventType.TEXT_MESSAGE_END,
            message_id=message_id
        ))
    
    elif event_type == "on_tool_start":
        tool_call_id = str(uuid.uuid4())
        tool_name = lg_event.get("name", "unknown_tool")
        yield encoder.encode(ToolCallStartEvent(
            type=EventType.TOOL_CALL_START,
            tool_call_id=tool_call_id,
            tool_call_name=tool_name
        ))
    
    elif event_type == "on_tool_end":
        tool_call_id = data.get("tool_call_id", str(uuid.uuid4()))
        output = data.get("output", "")
        yield encoder.encode(ToolCallEndEvent(
            type=EventType.TOOL_CALL_END,
            tool_call_id=tool_call_id
        ))
        yield encoder.encode(ToolCallResultEvent(
            type=EventType.TOOL_CALL_RESULT,
            tool_call_id=tool_call_id,
            result=str(output)
        ))
```

## Full LangGraph Endpoint

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from langgraph.graph import StateGraph
from ag_ui.core import *
from ag_ui.encoder import EventEncoder
import uuid

app = FastAPI()

# Your LangGraph workflow
workflow = StateGraph(...)
graph = workflow.compile()

@app.post("/jarvis")
async def jarvis_endpoint(input_data: RunAgentInput):
    encoder = EventEncoder()
    
    async def stream():
        # Start run
        yield encoder.encode(RunStartedEvent(
            type=EventType.RUN_STARTED,
            thread_id=input_data.thread_id,
            run_id=input_data.run_id
        ))
        
        # Prepare LangGraph input
        lg_input = {
            "messages": [
                {"role": m.role, "content": m.content}
                for m in input_data.messages
            ],
            "state": input_data.state
        }
        
        message_id = str(uuid.uuid4())
        message_started = False
        
        try:
            # Stream LangGraph events
            async for event in graph.astream_events(lg_input, version="v2"):
                async for ag_event in translate_langgraph_event(
                    event, encoder, message_id
                ):
                    yield ag_event
            
            # End run
            yield encoder.encode(RunFinishedEvent(
                type=EventType.RUN_FINISHED,
                thread_id=input_data.thread_id,
                run_id=input_data.run_id
            ))
            
        except Exception as e:
            yield encoder.encode(RunErrorEvent(
                type=EventType.RUN_ERROR,
                message=str(e),
                code="INTERNAL_ERROR"
            ))
    
    return StreamingResponse(stream(), media_type="text/event-stream")
```

## Async Queue Pattern

For complex workflows where events are generated from multiple sources:

```python
import asyncio
from typing import AsyncGenerator

class AGUIEventQueue:
    """Queue-based event handling for complex workflows."""
    
    def __init__(self, encoder: EventEncoder):
        self.encoder = encoder
        self.queue: asyncio.Queue = asyncio.Queue()
        self._finished = False
    
    async def emit(self, event: BaseEvent):
        """Add event to queue."""
        await self.queue.put(event)
    
    async def finish(self):
        """Signal completion."""
        self._finished = True
        await self.queue.put(None)  # Sentinel
    
    async def stream(self) -> AsyncGenerator[str, None]:
        """Yield encoded events from queue."""
        while True:
            event = await self.queue.get()
            if event is None:
                break
            yield self.encoder.encode(event)

# Usage in endpoint
@app.post("/jarvis")
async def jarvis_endpoint(input_data: RunAgentInput):
    encoder = EventEncoder()
    event_queue = AGUIEventQueue(encoder)
    
    async def run_workflow():
        await event_queue.emit(RunStartedEvent(...))
        
        # Run LangGraph workflow
        async for lg_event in graph.astream_events(input_data, version="v2"):
            # Translate and emit
            ag_events = translate_langgraph_event(lg_event, ...)
            for event in ag_events:
                await event_queue.emit(event)
        
        await event_queue.emit(RunFinishedEvent(...))
        await event_queue.finish()
    
    # Start workflow in background
    asyncio.create_task(run_workflow())
    
    # Stream from queue
    return StreamingResponse(
        event_queue.stream(),
        media_type="text/event-stream"
    )
```

## State Synchronization with LangGraph

Sync LangGraph state with frontend via STATE_DELTA:

```python
import jsonpatch

class StatefulLangGraphBridge:
    """Bridge LangGraph state to AG-UI state events."""
    
    def __init__(self, encoder: EventEncoder):
        self.encoder = encoder
        self.last_state = {}
    
    async def sync_state(self, new_state: dict) -> AsyncGenerator[str, None]:
        """Emit state delta for changed state."""
        if not self.last_state:
            # First sync - send snapshot
            yield self.encoder.encode(StateSnapshotEvent(
                type=EventType.STATE_SNAPSHOT,
                snapshot=new_state
            ))
        else:
            # Subsequent syncs - send delta
            patch = jsonpatch.make_patch(self.last_state, new_state)
            if patch.patch:
                yield self.encoder.encode(StateDeltaEvent(
                    type=EventType.STATE_DELTA,
                    delta=patch.patch
                ))
        
        self.last_state = new_state.copy()

# Usage in workflow
state_bridge = StatefulLangGraphBridge(encoder)

async for event in graph.astream_events(input_data, version="v2"):
    # Translate event
    async for ag_event in translate_langgraph_event(event, ...):
        yield ag_event
    
    # Sync state after each event
    if event.get("event") == "on_chain_end":
        current_state = event.get("data", {}).get("output", {})
        async for state_event in state_bridge.sync_state(current_state):
            yield state_event
```

## TypeScript Official Adapter

For TypeScript deployments, use the official adapter:

```typescript
import { LangGraphAgent } from "@ag-ui/langgraph";

const agent = new LangGraphAgent({
  graphId: "jarvis-lifelog",
  deploymentUrl: "https://your-deployment.langchain.app",
  langsmithApiKey: process.env.LANGSMITH_API_KEY
});

// Use with CopilotKit runtime
const runtime = new CopilotRuntime({
  agents: {
    jarvis: agent
  }
});
```

## Tool Call Tracking

Track tool calls across LangGraph events:

```python
class ToolCallTracker:
    """Track tool calls for proper AG-UI event sequencing."""
    
    def __init__(self):
        self.active_tools: Dict[str, str] = {}  # run_id -> tool_call_id
    
    def start_tool(self, lg_event: dict) -> str:
        """Generate and track tool_call_id."""
        run_id = lg_event.get("run_id", "")
        tool_call_id = str(uuid.uuid4())
        self.active_tools[run_id] = tool_call_id
        return tool_call_id
    
    def end_tool(self, lg_event: dict) -> Optional[str]:
        """Get and remove tracked tool_call_id."""
        run_id = lg_event.get("run_id", "")
        return self.active_tools.pop(run_id, None)

# Usage
tracker = ToolCallTracker()

if event_type == "on_tool_start":
    tool_call_id = tracker.start_tool(lg_event)
    yield encoder.encode(ToolCallStartEvent(..., tool_call_id=tool_call_id))

elif event_type == "on_tool_end":
    tool_call_id = tracker.end_tool(lg_event)
    if tool_call_id:
        yield encoder.encode(ToolCallEndEvent(..., tool_call_id=tool_call_id))
```
