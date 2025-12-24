# Frontend Integration

## CopilotKit Setup

### Installation

```bash
npm install @copilotkit/react-core @copilotkit/react-ui
```

### Provider Configuration

```tsx
import { CopilotKit } from "@copilotkit/react-core";
import { CopilotPopup } from "@copilotkit/react-ui";
import "@copilotkit/react-ui/styles.css";

function App() {
  return (
    <CopilotKit 
      runtimeUrl="/api/copilotkit"
      agent="jarvis_agent"
    >
      <YourApp />
      <CopilotPopup 
        labels={{ title: "Jarvis Assistant" }}
        shortcut="cmd+k"
      />
    </CopilotKit>
  );
}
```

### API Route (Next.js)

```typescript
// app/api/copilotkit/route.ts
import { CopilotRuntime, copilotRuntimeNextJSAppRouterEndpoint } from "@copilotkit/runtime";
import { HttpAgent } from "@ag-ui/client";

const runtime = new CopilotRuntime({
  agents: {
    jarvis_agent: new HttpAgent({
      url: process.env.JARVIS_AGENT_URL || "http://localhost:8000/jarvis"
    })
  }
});

export const POST = async (req: NextRequest) => {
  const { handleRequest } = copilotRuntimeNextJSAppRouterEndpoint({
    runtime,
    endpoint: "/api/copilotkit"
  });
  return handleRequest(req);
};
```

## useCoAgent Hook

Bidirectional state synchronization with agent:

```tsx
import { useCoAgent } from "@copilotkit/react-core";

interface JarvisState {
  query: string;
  results: LifelogEntry[];
  loading: boolean;
  error: string | null;
}

function SearchPanel() {
  const { 
    state,      // Current synced state
    setState,   // Update state (sends to agent)
    running,    // Is agent running?
    start,      // Trigger agent run
    stop        // Cancel current run
  } = useCoAgent<JarvisState>({
    name: "jarvis_agent",
    initialState: {
      query: "",
      results: [],
      loading: false,
      error: null
    }
  });

  const handleSearch = () => {
    setState({ ...state, query: inputValue });
    start();  // Trigger agent with updated state
  };

  return (
    <div>
      <input 
        value={state.query}
        onChange={(e) => setState({ ...state, query: e.target.value })}
      />
      <button onClick={handleSearch} disabled={running}>
        {running ? "Searching..." : "Search"}
      </button>
      
      {state.loading && <LoadingSpinner />}
      {state.error && <ErrorMessage>{state.error}</ErrorMessage>}
      
      <ResultsList results={state.results} />
    </div>
  );
}
```

## useCoAgentStateRender Hook

Render UI based on agent state (generative UI):

```tsx
import { useCoAgentStateRender } from "@copilotkit/react-core";

function LifelogResults() {
  useCoAgentStateRender<JarvisState>({
    name: "jarvis_agent",
    render: ({ state, status }) => {
      // Render based on current state
      if (status === "running" && state.loading) {
        return <SearchingAnimation query={state.query} />;
      }
      
      if (state.results.length > 0) {
        return <LifelogTimeline entries={state.results} />;
      }
      
      if (state.error) {
        return <ErrorDisplay message={state.error} />;
      }
      
      return null;  // No custom render
    }
  });

  return null;  // Hook handles rendering
}
```

## useCopilotAction Hook

Define frontend-callable tools:

```tsx
import { useCopilotAction } from "@copilotkit/react-core";

function LifelogViewer() {
  const [selectedEntry, setSelectedEntry] = useState<LifelogEntry | null>(null);

  useCopilotAction({
    name: "show_lifelog_detail",
    description: "Display detailed view of a lifelog entry",
    parameters: [
      {
        name: "entry_id",
        type: "string",
        description: "The ID of the entry to display",
        required: true
      }
    ],
    // Render while tool executes
    render: ({ status, args }) => {
      if (status === "executing") {
        return <EntryPreviewSkeleton />;
      }
      return <EntryPreview id={args.entry_id} />;
    },
    // Handler executes when agent calls tool
    handler: async ({ entry_id }) => {
      const entry = await fetchEntry(entry_id);
      setSelectedEntry(entry);
      return `Showing entry: ${entry.title}`;
    }
  });

  return <EntryDetailModal entry={selectedEntry} />;
}
```

## useFrontendTool Hook

Streaming tool results with progressive rendering:

```tsx
import { useFrontendTool } from "@copilotkit/react-core";

function TimelineRenderer() {
  useFrontendTool({
    name: "render_timeline",
    description: "Render lifelog entries as interactive timeline",
    parameters: [
      {
        name: "entries",
        type: "object[]",
        description: "Array of lifelog entries"
      },
      {
        name: "view_mode",
        type: "string",
        enum: ["day", "week", "month"]
      }
    ],
    render: ({ status, args }) => {
      // Progressive rendering as args stream in
      const entries = args.entries || [];
      const mode = args.view_mode || "day";
      
      return (
        <Timeline 
          entries={entries} 
          mode={mode}
          loading={status === "executing"}
        />
      );
    },
    handler: ({ entries, view_mode }) => {
      // Optional side effects
      analytics.track("timeline_rendered", { count: entries.length });
    }
  });
}
```

## Human-in-the-Loop Patterns

### renderAndWait

Pause agent until user responds:

```tsx
useCopilotAction({
  name: "delete_lifelog_entry",
  description: "Delete a lifelog entry (requires confirmation)",
  parameters: [
    { name: "entry_id", type: "string", required: true },
    { name: "entry_title", type: "string", required: true }
  ],
  renderAndWait: ({ args, handler }) => (
    <ConfirmationDialog
      title="Confirm Deletion"
      message={`Delete "${args.entry_title}"? This cannot be undone.`}
      onConfirm={() => handler({ approved: true })}
      onCancel={() => handler({ approved: false })}
    />
  ),
  handler: async ({ entry_id }, response) => {
    if (!response?.approved) {
      return "Deletion cancelled by user.";
    }
    await deleteEntry(entry_id);
    return `Deleted entry: ${entry_id}`;
  }
});
```

### useHumanInTheLoop

Handle interrupt events from backend:

```tsx
import { useHumanInTheLoop } from "@copilotkit/react-core";

function ApprovalHandler() {
  useHumanInTheLoop({
    name: "approve_summary",
    render: ({ args, status, respond }) => {
      if (status !== "waiting") return null;
      
      return (
        <ApprovalCard>
          <h3>Review Summary</h3>
          <blockquote>{args.summary}</blockquote>
          <div>
            <button onClick={() => respond({ action: "approve" })}>
              ✓ Approve
            </button>
            <button onClick={() => respond({ action: "reject" })}>
              ✗ Reject
            </button>
            <button onClick={() => respond({ action: "edit", edited: "..." })}>
              ✎ Edit
            </button>
          </div>
        </ApprovalCard>
      );
    }
  });

  return null;
}
```

## Context Providers

Provide additional context to agent:

```tsx
import { useCopilotReadable } from "@copilotkit/react-core";

function ContextProvider() {
  const { user } = useAuth();
  const { currentProject } = useProject();

  // Make user context available to agent
  useCopilotReadable({
    description: "Current user information",
    value: {
      userId: user.id,
      name: user.name,
      timezone: user.timezone,
      preferences: user.preferences
    }
  });

  // Make project context available
  useCopilotReadable({
    description: "Current project context",
    value: {
      projectId: currentProject.id,
      projectName: currentProject.name,
      recentFiles: currentProject.recentFiles
    }
  });

  return null;
}
```

## Custom Event Handling

Handle CUSTOM events from backend:

```tsx
import { useCopilotCustomEvent } from "@copilotkit/react-core";

function CustomEventHandler() {
  const { addToast } = useToast();

  useCopilotCustomEvent({
    name: "INTERRUPT_APPROVAL_REQUIRED",
    handler: ({ value }) => {
      // Handle custom interrupt
      addToast({
        type: "warning",
        message: value.message,
        actions: value.options
      });
    }
  });

  useCopilotCustomEvent({
    name: "ACTIVITY_UPDATE",
    handler: ({ value }) => {
      // Handle activity updates
      console.log("Agent activity:", value.activity);
    }
  });
}
```

## UI Components

### CopilotPopup

Floating chat interface:

```tsx
<CopilotPopup
  labels={{
    title: "Jarvis",
    placeholder: "Ask about your lifelogs...",
    initial: "How can I help you today?"
  }}
  shortcut="cmd+k"
  defaultOpen={false}
  clickOutsideToClose={true}
/>
```

### CopilotSidebar

Side panel interface:

```tsx
<CopilotSidebar
  labels={{ title: "Jarvis Assistant" }}
  defaultOpen={true}
>
  <YourMainContent />
</CopilotSidebar>
```

### CopilotChat

Inline chat component:

```tsx
<div className="chat-container">
  <CopilotChat
    labels={{ placeholder: "Search lifelogs..." }}
    className="h-full"
  />
</div>
```

## Styling

```css
/* Override CopilotKit styles */
:root {
  --copilot-kit-primary-color: #0066cc;
  --copilot-kit-background-color: #ffffff;
  --copilot-kit-text-color: #1a1a1a;
  --copilot-kit-border-radius: 8px;
}

/* Custom message styles */
.copilot-message-assistant {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}
```
