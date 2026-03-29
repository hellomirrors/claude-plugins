# hellomirrors Claude Plugins

Plugin marketplace for the hellomirrors team. Provides shared skills, agents, and commands for Claude Code.

## Plugins

| Plugin | Description |
|--------|-------------|
| **hm-webapp-stack** | Opinionated web-app scaffolding: Next.js, shadcn/ui, Tailwind CSS 4, Ultracite/Biome, Zod, TanStack Query/Form |

## Setup

Add this to your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "extraKnownMarketplaces": {
    "hellomirrors": {
      "source": {
        "source": "github",
        "repo": "hellomirrors/claude-plugins"
      }
    }
  },
  "enabledPlugins": {
    "hm-webapp-stack@hellomirrors": true
  }
}
```

Or install interactively via `/plugins` in Claude Code.
