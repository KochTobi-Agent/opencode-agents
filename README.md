# OpenCode Agents

A collection of agent definitions for different tasks in OpenCode.

## Repository Structure

```
opencode-agents/
├── README.md              # This file
├── opencode.jsonc         # OpenCode configuration with MCP servers
├── agents/                # Agent definitions (markdown)
│   └── research.md       # Research agent for Context7
└── research/              # Output directory for collected research
```

## Agents

### Research Agent

A specialized agent that collects up-to-date library and framework documentation from Context7 and saves it locally.

**Location:** `agents/research.md`

**Usage:** Invoke with `@research` in OpenCode

## Setup

### 1. Configure MCP Server

Copy `opencode.jsonc` to your project root or global config location (`~/.config/opencode/`).

To use Context7 with API key (for higher rate limits):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "context7": {
      "type": "remote",
      "url": "https://mcp.context7.com/mcp",
      "headers": {
        "CONTEXT7_API_KEY": "{env:CONTEXT7_API_KEY}"
      }
    }
  }
}
```

Authenticate with:
```bash
opencode mcp auth context7
```

### 2. Install Agents

Copy agent markdown files to your OpenCode agents directory:

- **Global:** `~/.config/opencode/agents/`
- **Per-project:** `.opencode/agents/`

For example:
```bash
cp agents/research.md ~/.config/opencode/agents/research.md
```

## Usage

When you need to research library or framework documentation, simply mention the research agent in your prompt:

```
@research I need to understand how to use Prisma with Next.js
```

The agent will:
1. Use Context7 MCP to fetch up-to-date documentation
2. Collect relevant information
3. Save the research as a markdown file in the `research/` directory
