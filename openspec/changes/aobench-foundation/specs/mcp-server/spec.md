# Spec Delta

## Purpose

A Model Context Protocol server (`packages/mcp`) that exposes bot control, state reads, chat, and wiki search as MCP tools, so AI assistants (Claude, Cursor, Kiro) can control bots and inspect game state directly without the harness CLI.

## ADDED Requirements

### Requirement: MCP server exposes get_state as a tool
The MCP server SHALL expose a `get_state` tool that accepts a `charname` parameter and returns the current WorldState JSON for that bot.

#### Scenario: AI assistant reads bot state
- **WHEN** an AI assistant calls `get_state(charname: "bot1")`
- **THEN** the tool returns the current WorldState including position, HP, skills, inventory, and nearby entities

### Requirement: MCP server exposes execute_action as a tool
The MCP server SHALL expose an `execute_action` tool that accepts a `charname` and an `action` object matching the BotAPI command schema, sends it via the gateway, and returns the result.

#### Scenario: AI assistant moves a bot
- **WHEN** an AI assistant calls `execute_action(charname: "bot1", action: {type: "walk", heading: "north"})`
- **THEN** the bot moves north and the tool returns `{"ok": true}`

### Requirement: MCP server exposes execute_code as a tool
The MCP server SHALL expose an `execute_code` tool that accepts a `charname` and a TypeScript code string, runs it in the same Bun sandbox used by the harness, and streams stdout back as the tool result.

#### Scenario: AI assistant runs a bot script
- **WHEN** an AI assistant calls `execute_code(charname: "bot1", code: "const state = await bot.getState(); console.log(state.hp);")`
- **THEN** the code executes against the live bot and the HP is returned as output

### Requirement: MCP server exposes get_chat as a tool
The MCP server SHALL expose a `get_chat` tool that accepts a `charname` and optional `limit`, returning recent chat messages seen by the bot.

#### Scenario: AI assistant reads chat history
- **WHEN** an AI assistant calls `get_chat(charname: "bot1", limit: 20)`
- **THEN** the tool returns the 20 most recent chat messages

### Requirement: MCP server exposes say as a tool
The MCP server SHALL expose a `say` tool that accepts `charname` and `message`, sending the message through the bot character.

#### Scenario: AI assistant sends a chat message
- **WHEN** an AI assistant calls `say(charname: "bot1", message: "hola!")`
- **THEN** the bot says "hola!" in game and the tool confirms success

### Requirement: MCP server exposes search_wiki as a tool
The MCP server SHALL expose a `search_wiki` tool that accepts a `query` string and searches the wiki markdown files, returning relevant excerpts about items, NPCs, spells, skills, or maps.

#### Scenario: AI assistant looks up an NPC
- **WHEN** an AI assistant calls `search_wiki(query: "lobo stats")`
- **THEN** the tool returns the NPC entry for "lobo" with its stats and drops

### Requirement: MCP server connects to the gateway
The MCP server SHALL connect to the gateway WebSocket to send actions and receive state, rather than connecting to BotAPI.bas directly.

#### Scenario: MCP server routes through gateway
- **WHEN** execute_action is called
- **THEN** the command is sent via the gateway's WebSocket protocol, not directly to port 4200
