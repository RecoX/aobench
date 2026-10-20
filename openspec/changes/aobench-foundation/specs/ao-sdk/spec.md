# Spec Delta

## Purpose

A TypeScript package that wraps the BotAPI TCP JSON interface into a clean, agent-facing API modelled after rs-sdk, so AI agents write scripts against typed methods without knowing anything about TCP sockets or the AO binary protocol.

## ADDED Requirements

### Requirement: SDK exposes an AOBot class
The SDK SHALL export an `AOBot` class instantiated with `{ server, port, charname }` that manages the TCP connection to BotAPI.bas and exposes all actions and state reads as async methods.

#### Scenario: AOBot connects successfully
- **WHEN** `await bot.connect()` is called with valid server/port/charname
- **THEN** the TCP connection is established and the bot is ready to issue commands

#### Scenario: Connection failure throws a descriptive error
- **WHEN** `await bot.connect()` cannot reach the server
- **THEN** it throws an error with the host, port, and reason

### Requirement: getState returns a typed WorldState
`await bot.getState()` SHALL send `{"action":"getState","charname":"..."}` and return the response as a fully typed `WorldState` TypeScript object.

#### Scenario: getState returns typed object
- **WHEN** `await bot.getState()` is called on a connected bot
- **THEN** the return value has `pos`, `hp`, `mana`, `stamina`, `gold`, `exp`, `level`, `clase`, `heading`, `skills`, `inventory`, `nearbyNpcs`, `nearbyPlayers`, and `statusEffects` with correct TypeScript types

### Requirement: Movement methods cover all 8 headings
The SDK SHALL expose `walk(heading)` accepting `"north" | "south" | "east" | "west" | "northeast" | "northwest" | "southeast" | "southwest"`, and a higher-level `walkTo(x, y)` that calls `walk` repeatedly along a path.

#### Scenario: walk sends correct heading
- **WHEN** `await bot.walk("north")` is called
- **THEN** `{"action":"walk","heading":"north"}` is sent and resolved

#### Scenario: walkTo moves step by step
- **WHEN** `await bot.walkTo(55, 60)` is called
- **THEN** the SDK issues sequential walk commands until the character reaches (55, 60) per getState

### Requirement: Combat methods map to all attack actions
The SDK SHALL expose `leftClick(dx, dy)`, `attack()`, and `castSpell(slot)`.

#### Scenario: leftClick then attack targets an NPC
- **WHEN** `await bot.leftClick(1, 0)` then `await bot.attack()` are called
- **THEN** the NPC at offset (1,0) is targeted and attacked

### Requirement: Skill and item methods cover all common actions
The SDK SHALL expose `useSkill(skillName)`, `useItem(slot)`, `equipItem(slot)`, `pickUp(dx, dy)`, `drop(slot, amount)`, `meditate()`, and `rest()`.

#### Scenario: useSkill sends work command
- **WHEN** `await bot.useSkill("Herrería")` is called
- **THEN** `{"action":"work","skill":"Herrería"}` is sent

### Requirement: Chat methods cover talk, yell, and whisper
The SDK SHALL expose `say(message)`, `yell(message)`, and `whisper(targetName, message)`.

#### Scenario: say sends talk command
- **WHEN** `await bot.say("hola!")` is called
- **THEN** `{"action":"talk","message":"hola!"}` is sent

### Requirement: waitTicks pauses execution
`await bot.waitTicks(n)` SHALL pause execution for `n × 600ms` (one ao-libre server tick).

#### Scenario: waitTicks delays execution
- **WHEN** `await bot.waitTicks(5)` is called
- **THEN** execution resumes after approximately 3000ms

### Requirement: runScript executes an agent function with cleanup
`await runScript(serverOpts, async (bot) => { ... })` SHALL connect the bot, run the provided async function, then cleanly disconnect regardless of success or error — matching rs-sdk's `runScript` pattern.

#### Scenario: runScript cleans up on error
- **WHEN** the script function throws
- **THEN** the bot disconnects and runScript rethrows the error

### Requirement: SDK ships full TypeScript types
The package SHALL export `WorldState`, `NearbyNpc`, `NearbyPlayer`, `InventorySlot`, `Skills`, and `Heading` types so agents get autocomplete and type safety.

#### Scenario: Types importable by agents
- **WHEN** an agent imports `{ WorldState }` from `@aobench/sdk`
- **THEN** the type is available with all fields correctly typed

### Requirement: SDK is published as @aobench/sdk
The package SHALL be published under `@aobench/sdk` so agents install it with `npm install @aobench/sdk`.

#### Scenario: Package installable by name
- **WHEN** `npm install @aobench/sdk` is run
- **THEN** the package installs successfully with its type definitions
