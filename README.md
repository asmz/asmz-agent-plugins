# asmz-agent-plugins

A marketplace of Claude Code plugins published by asmz.

## Add this marketplace

```bash
/plugin marketplace add asmz/asmz-agent-plugins
```

---

## [maestro-flutter-agent](./plugins/maestro-flutter-agent)

An agent skill that generates, runs, debugs, and fixes Maestro test flows (YAML) for Flutter mobile apps. Given a test scenario in natural language, it analyzes Flutter widget code to produce the matching Maestro YAML flow, then autonomously executes and verifies it via the Maestro MCP.

### Installation

```bash
/plugin install maestro-flutter-agent@asmz-agent-plugins
```

### Requirements

- [Maestro CLI](https://docs.maestro.dev/) installed and available on `PATH` (the plugin starts `maestro mcp` automatically as its bundled MCP server)
- A Flutter app project with readable Dart sources (local project)
- An iOS simulator, Android emulator, or physical device available when running flows

### First-time install

The plugin bundles a Maestro MCP server (`maestro mcp`) and Claude Code will ask for one-time approval the first time you install the plugin. Approve it to enable the `mcp__maestro__*` tools used by the agent skill.

---

## [maestro-react-native-agent](./plugins/maestro-react-native-agent)

An agent skill that generates, runs, debugs, and fixes Maestro test flows (YAML) for React Native mobile apps. Given a test scenario in natural language, it reads React Native component code (JSX/TSX) and produces the matching Maestro YAML flow, then autonomously executes and verifies it via the Maestro MCP. It auto-detects whether the project is Expo or bare React Native and runs accordingly — defaulting to Expo Go for Expo projects (with a Metro server availability check) and to a native binary (`.app` / `.apk`) for bare projects.

### Installation

```bash
/plugin install maestro-react-native-agent@asmz-agent-plugins
```

### Requirements

- [Maestro CLI](https://docs.maestro.dev/) installed and available on `PATH` (the plugin starts `maestro mcp` automatically as its bundled MCP server)
- A React Native app project with readable JS/TS sources (local project). Expo managed / Expo dev build / bare React Native are all supported
- For Expo Go execution: the Expo Go app installed on the target simulator/emulator, and `npx expo start` runnable as the development server
- For native-binary execution: a buildable `.app` (iOS) or `.apk` (Android)
- An iOS simulator, Android emulator, or physical device available when running flows

### First-time install

The plugin bundles a Maestro MCP server (`maestro mcp`) and Claude Code will ask for one-time approval the first time you install the plugin. Approve it to enable the `mcp__maestro__*` tools used by the agent skill.

---

## License

[MIT](./LICENSE)
