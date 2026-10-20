# Spec Delta

## Purpose

A CLI that orchestrates a complete benchmark run against the ao-libre bot server: provisioning a character, injecting wiki docs and ao-sdk as agent context, running an LLM agent in a sandbox, measuring XP rate, and submitting results to the leaderboard.

## ADDED Requirements

### Requirement: Harness provisions a fresh bot character per run
The harness SHALL create a new account and character on the bot server before each run and remove it after, so runs start from a clean slate.

#### Scenario: Fresh character exists at run start
- **WHEN** a benchmark run starts
- **THEN** a new bot account and character with a generated name is logged in to the bot server

#### Scenario: Character cleaned up after run
- **WHEN** a benchmark run ends (success or failure)
- **THEN** the character's account and charfile are deleted from the bot server

### Requirement: Harness injects wiki docs and SDK as agent context
Before running agent code, the harness SHALL provide the full wiki markdown files and ao-sdk README and type definitions as the agent's context.

#### Scenario: Agent context includes all wiki files
- **WHEN** the harness starts a run
- **THEN** the agent receives `items.md`, `npcs.md`, `spells.md`, `skills.md`, and `maps.md` as context

#### Scenario: Agent context includes SDK reference
- **WHEN** the harness starts a run
- **THEN** the agent receives the ao-sdk README and exported TypeScript types as context

### Requirement: Harness executes agent TypeScript in a Bun subprocess sandbox
The harness SHALL run agent-produced TypeScript in a Bun subprocess that can only import `@aobench/sdk` and standard library modules. Filesystem access, arbitrary network calls, and process spawning SHALL be blocked.

#### Scenario: Agent script runs against live bot server
- **WHEN** the agent produces a TypeScript snippet and the harness executes it
- **THEN** the script runs and its `AOBot` calls are visible as actions in the game

#### Scenario: Unauthorized import is blocked
- **WHEN** agent code tries to import `fs` or make a direct `fetch` call
- **THEN** the sandbox blocks it and the run records a sandbox violation

### Requirement: Harness measures max XP rate in 15-second windows
The harness SHALL poll `bot.getState()` at fine intervals, record XP totals, and report the maximum XP gained in any 15-second window as the primary benchmark score.

#### Scenario: XP rate computed from polling
- **WHEN** a run ends
- **THEN** the score equals the maximum XP delta observed across any 15-second window during the run

### Requirement: Harness loads task definitions from YAML
The harness SHALL read task config from `tasks/<name>.yaml` specifying time limit, starting map/position, and scoring metric. The first task SHALL be `magia-xp` (maximize Magia skill XP in 15 minutes).

#### Scenario: Task parameters applied
- **WHEN** `--task magia-xp` is passed
- **THEN** the run enforces the task's time limit and spawns the character in the configured starting location

### Requirement: Harness supports OpenAI and Anthropic providers
The harness SHALL accept a `--model <name>` flag and route inference to OpenAI (`gpt-*`) or Anthropic (`claude-*`) based on the model name prefix.

#### Scenario: OpenAI model runs as agent
- **WHEN** `--model gpt-4o` is passed
- **THEN** the harness uses the OpenAI API for agent inference

#### Scenario: Anthropic model runs as agent
- **WHEN** `--model claude-sonnet-4-5` is passed
- **THEN** the harness uses the Anthropic API for agent inference

### Requirement: Harness submits results to the leaderboard API
After each run the harness SHALL POST the result to the leaderboard including model name, task name, score, duration, and run log.

#### Scenario: Result submitted after successful run
- **WHEN** a run completes
- **THEN** the harness POSTs to the leaderboard and prints the returned run ID
