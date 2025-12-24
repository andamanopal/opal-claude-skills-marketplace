# Jarvis Skills Marketplace

Custom Claude Code plugins and skills for Project Jarvis - an AI-powered personal assistant with lifelog indexing and real-time voice interaction.

## Installation

Add this marketplace to Claude Code:

```bash
/plugin marketplace add YOUR_USERNAME/jarvis-skills-marketplace
```

Then install individual plugins:

```bash
/plugin install ag-ui-expert@jarvis-skills
```

## Available Plugins

### ag-ui-expert

Expert-level AG-UI protocol implementation for building real-time agent-user interfaces.

**Features:**
- SSE-based agent streaming patterns
- LangGraph-to-frontend integration
- CopilotKit hooks and setup
- Bidirectional state sync with JSON Patch (RFC 6902)
- Human-in-the-loop patterns
- Security hardening and production deployment

**Triggers:**
- Implementing SSE streaming for agents
- Setting up CopilotKit with custom backends
- Building real-time agent UIs
- State synchronization patterns
- AG-UI protocol questions

## Repository Structure

```
jarvis-skills-marketplace/
├── .claude-plugin/
│   └── marketplace.json          # Marketplace catalog
├── plugins/
│   └── ag-ui-expert/
│       ├── .claude-plugin/
│       │   └── plugin.json       # Plugin manifest
│       └── skills/
│           └── ag-ui-expert/
│               ├── SKILL.md      # Main skill file
│               └── references/   # Detailed documentation
│                   ├── python-sdk.md
│                   ├── langgraph-integration.md
│                   ├── state-management.md
│                   ├── frontend-integration.md
│                   └── security-deployment.md
└── README.md
```

## Adding New Skills

1. Create a new plugin directory under `plugins/`
2. Add `.claude-plugin/plugin.json` with plugin metadata
3. Add skills in `skills/` directory with `SKILL.md`
4. Update `.claude-plugin/marketplace.json` with the new plugin entry

## License

MIT
