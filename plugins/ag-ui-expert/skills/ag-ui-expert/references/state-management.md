# State Management

## Overview

AG-UI uses bidirectional state synchronization between frontend and backend. Frontend sends state in `RunAgentInput.state`, backend emits `STATE_SNAPSHOT` and `STATE_DELTA` events.

## JSON Patch (RFC 6902)

State deltas use JSON Patch format for efficient incremental updates.

### Operations

| Operation | Description | Example |
|-----------|-------------|---------|
| `add` | Add value at path | `{"op": "add", "path": "/results/-", "value": {...}}` |
| `remove` | Remove value at path | `{"op": "remove", "path": "/results/0"}` |
| `replace` | Replace value at path | `{"op": "replace", "path": "/loading", "value": false}` |
| `move` | Move value from→to | `{"op": "move", "from": "/temp", "path": "/final"}` |
| `copy` | Copy value from→to | `{"op": "copy", "from": "/template", "path": "/new"}` |
| `test` | Assert value before applying | `{"op": "test", "path": "/version", "value": 2}` |

### Path Syntax (JSON Pointer RFC 6901)

| Path | Meaning |
|------|---------|
| `/` | Root object |
| `/foo` | Property "foo" |
| `/foo/0` | First element of array "foo" |
| `/foo/-` | Append to array "foo" |
| `/foo/bar~1baz` | Property "bar/baz" (/ escaped as ~1) |
| `/foo/bar~0baz` | Property "bar~baz" (~ escaped as ~0) |

### Python jsonpatch Library

```bash
pip install jsonpatch
```

```python
import jsonpatch

# Generate patch from two states
original = {"results": [], "loading": True}
modified = {"results": [{"id": 1}], "loading": False}

patch = jsonpatch.make_patch(original, modified)
print(patch.patch)
# [
#   {"op": "add", "path": "/results/0", "value": {"id": 1}},
#   {"op": "replace", "path": "/loading", "value": false}
# ]

# Apply patch to document
result = jsonpatch.apply_patch(original, patch.patch)
```

## State Events

### STATE_SNAPSHOT

Send full state, typically at run start:

```python
yield encoder.encode(StateSnapshotEvent(
    type=EventType.STATE_SNAPSHOT,
    snapshot={
        "query": "",
        "results": [],
        "loading": False,
        "error": None,
        "selectedId": None
    }
))
```

### STATE_DELTA

Send incremental updates during run:

```python
# Set loading
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[{"op": "replace", "path": "/loading", "value": True}]
))

# Add result to array
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[{"op": "add", "path": "/results/-", "value": {"id": 1, "text": "..."}}]
))

# Multiple operations in one delta
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[
        {"op": "replace", "path": "/loading", "value": False},
        {"op": "replace", "path": "/query", "value": "meeting notes"}
    ]
))
```

## Bidirectional Flow

```
┌─────────────────────────────────────────────────────────┐
│                       Frontend                          │
│  ┌─────────────────────────────────────────────────┐   │
│  │  state = {results: [...], loading: false}       │   │
│  └─────────────────────────────────────────────────┘   │
│         │                              ▲                │
│         │ RunAgentInput.state          │ Apply patch    │
│         ▼                              │                │
└─────────────────────────────────────────────────────────┘
          │                              │
          │ POST /agent                  │ SSE stream
          ▼                              │
┌─────────────────────────────────────────────────────────┐
│                       Backend                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  1. Receive state from input                    │   │
│  │  2. Process workflow                            │   │
│  │  3. Emit STATE_SNAPSHOT (initial)               │   │
│  │  4. Emit STATE_DELTA (incremental)              │──┘
│  └─────────────────────────────────────────────────┘
└─────────────────────────────────────────────────────────┘
```

## State Manager Pattern

```python
import jsonpatch
from typing import Any, Dict, List

class JarvisStateManager:
    """Manage state synchronization for AG-UI."""
    
    def __init__(self, encoder: EventEncoder):
        self.encoder = encoder
        self.current_state: Dict[str, Any] = {}
        self.initialized = False
    
    def initialize(self, initial_state: Dict[str, Any]) -> str:
        """Send initial state snapshot."""
        self.current_state = initial_state.copy()
        self.initialized = True
        return self.encoder.encode(StateSnapshotEvent(
            type=EventType.STATE_SNAPSHOT,
            snapshot=initial_state
        ))
    
    def update(self, updates: Dict[str, Any]) -> str:
        """Apply updates and emit delta."""
        new_state = {**self.current_state, **updates}
        patch = jsonpatch.make_patch(self.current_state, new_state)
        
        if patch.patch:
            self.current_state = new_state
            return self.encoder.encode(StateDeltaEvent(
                type=EventType.STATE_DELTA,
                delta=patch.patch
            ))
        return ""
    
    def append_result(self, result: Any) -> str:
        """Append to results array."""
        return self.encoder.encode(StateDeltaEvent(
            type=EventType.STATE_DELTA,
            delta=[{"op": "add", "path": "/results/-", "value": result}]
        ))
    
    def set_loading(self, loading: bool) -> str:
        """Set loading state."""
        return self.encoder.encode(StateDeltaEvent(
            type=EventType.STATE_DELTA,
            delta=[{"op": "replace", "path": "/loading", "value": loading}]
        ))
    
    def set_error(self, error: str | None) -> str:
        """Set error state."""
        return self.encoder.encode(StateDeltaEvent(
            type=EventType.STATE_DELTA,
            delta=[{"op": "replace", "path": "/error", "value": error}]
        ))

# Usage
state_manager = JarvisStateManager(encoder)

async def stream():
    yield encoder.encode(RunStartedEvent(...))
    
    # Initialize state
    yield state_manager.initialize({
        "query": input_data.state.get("query", ""),
        "results": [],
        "loading": True,
        "error": None
    })
    
    # Process and update
    for result in search_results:
        yield state_manager.append_result(result)
    
    yield state_manager.set_loading(False)
    yield encoder.encode(RunFinishedEvent(...))
```

## Common State Patterns

### Loading States

```python
# Start loading
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[
        {"op": "replace", "path": "/loading", "value": True},
        {"op": "replace", "path": "/error", "value": None}
    ]
))

# Finish loading
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[{"op": "replace", "path": "/loading", "value": False}]
))
```

### Streaming Results

```python
# Progressive result streaming
for i, result in enumerate(results):
    yield encoder.encode(StateDeltaEvent(
        type=EventType.STATE_DELTA,
        delta=[{"op": "add", "path": "/results/-", "value": result}]
    ))
    # Optional: update progress
    yield encoder.encode(StateDeltaEvent(
        type=EventType.STATE_DELTA,
        delta=[{"op": "replace", "path": "/progress", "value": (i+1)/len(results)}]
    ))
```

### Nested State Updates

```python
# Update nested object
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[
        {"op": "replace", "path": "/user/preferences/theme", "value": "dark"},
        {"op": "add", "path": "/user/history/-", "value": {"query": "...", "timestamp": "..."}}
    ]
))
```

### Clear and Reset

```python
# Clear results
yield encoder.encode(StateDeltaEvent(
    type=EventType.STATE_DELTA,
    delta=[{"op": "replace", "path": "/results", "value": []}]
))

# Full reset via snapshot
yield encoder.encode(StateSnapshotEvent(
    type=EventType.STATE_SNAPSHOT,
    snapshot={"query": "", "results": [], "loading": False, "error": None}
))
```

## Security: Prototype Pollution Prevention

Always validate JSON Patch paths before applying:

```python
FORBIDDEN_PATHS = ["__proto__", "constructor", "prototype"]

def validate_patch(patch: List[dict]) -> bool:
    """Check for prototype pollution attempts."""
    for op in patch:
        path = op.get("path", "")
        from_path = op.get("from", "")
        
        for forbidden in FORBIDDEN_PATHS:
            if forbidden in path or forbidden in from_path:
                return False
    return True

# Usage
patch = jsonpatch.make_patch(old_state, new_state)
if not validate_patch(patch.patch):
    raise ValueError("Invalid patch: potential prototype pollution")
```
