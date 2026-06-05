# asmz-claude-plugins

A marketplace of Claude Code plugins published by asmz.

## Plugins

### [maestro-flutter-agent](./plugins/maestro-flutter-agent)

An agent skill that generates, runs, debugs, and fixes Maestro test flows (YAML) for Flutter mobile apps. Given a test scenario in natural language, it analyzes Flutter widget code to produce the matching Maestro YAML flow, then autonomously executes and verifies it via the Maestro MCP.

## Installation

Add this marketplace and install the plugin:

```bash
# Add the marketplace
/plugin marketplace add asmz/asmz-claude-plugins

# Install the plugin
/plugin install maestro-flutter-agent@asmz-claude-plugins
```

## Requirements

- [Maestro CLI](https://docs.maestro.dev/) installed and available on `PATH` (the plugin starts `maestro mcp` automatically as its bundled MCP server)
- A Flutter app project with readable Dart sources (local project)
- An iOS simulator, Android emulator, or physical device available when running flows

## First-time install

The plugin bundles a Maestro MCP server (`maestro mcp`) and Claude Code will ask for one-time approval the first time you install the plugin. Approve it to enable the `mcp__maestro__*` tools used by the agent skill.

## License

[MIT](./LICENSE)
