# Spec Delta

## Purpose

A `lab_log.md` file in each bot directory that the LLM agent can read and append to during a run, enabling persistent observations, strategy notes, and discoveries that survive between script iterations and inform future agent decisions.

## ADDED Requirements

### Requirement: Each bot directory contains a lab_log.md
The `create-bot` scaffold SHALL include a `lab_log.md` with a header and instructions explaining its purpose to the agent.

#### Scenario: lab_log.md created with scaffold
- **WHEN** `bun create-bot mybot` is run
- **THEN** `bots/mybot/lab_log.md` exists with a header and placeholder content

### Requirement: Harness injects lab_log.md as agent context
The harness SHALL read the bot's `lab_log.md` and include it in the agent context alongside wiki docs and SDK reference before each run.

#### Scenario: Lab log included in agent context
- **WHEN** the harness starts a run for a bot with an existing lab_log.md
- **THEN** the lab_log contents appear in the agent's context

### Requirement: SDK exposes appendLabLog for writing observations
`await bot.appendLabLog(text)` SHALL append a timestamped entry to `lab_log.md` in the bot's directory. This is the only write operation permitted to the sandbox.

#### Scenario: Agent appends a note
- **WHEN** `await bot.appendLabLog("Level 12 Magia: best XP at lobos en mapa 1 pos (50,60)")` is called
- **THEN** a timestamped entry is appended to `bots/<charname>/lab_log.md`

#### Scenario: appendLabLog is available in the sandbox
- **WHEN** agent code running in the Bun subprocess calls `bot.appendLabLog(...)`
- **THEN** the call succeeds — it is explicitly allowed despite the filesystem write restriction on all other paths

### Requirement: lab_log.md is preserved between runs
The harness SHALL NOT clear or truncate `lab_log.md` between runs, so accumulated notes from previous runs remain available to future agents.

#### Scenario: Notes from previous run available in next run
- **WHEN** a second run starts for the same bot
- **THEN** the agent context includes notes appended during the previous run
