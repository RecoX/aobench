# Spec Delta

## Purpose

A fork of ao-libre/ao-server running in isolation from production, exclusively for AI agent bot accounts, with anticheat disabled, a configurable XP multiplier, and no email validation requirement.

## ADDED Requirements

### Requirement: Bot server is a fork of ao-libre/ao-server
The bot server SHALL be derived from `ao-libre/ao-server` (AGPL), hosted in `recox/ao-libre-server`, with bot-specific configuration and the new `BotAPI.bas` module added on top of the unmodified upstream codebase.

#### Scenario: Server starts from the fork
- **WHEN** the ao-libre server fork is compiled and started
- **THEN** it listens on its configured game port and the BotAPI port 4200, and accepts connections without affecting any other environment

### Requirement: Bot server runs in isolation
The bot server SHALL use its own character files, account files, and world state, sharing nothing with production or any other environment.

#### Scenario: Character created on bot server stays isolated
- **WHEN** a bot account and character are created on the bot server
- **THEN** no corresponding change occurs in any other environment's data

### Requirement: Anticheat is disabled in bot mode
The bot server SHALL compile and run with the `DEBUGGING=1` conditional flag (already supported by ao-libre) or a new `BOT_SERVER=1` flag that disables all anticheat checks, so bot clients can connect and act freely.

#### Scenario: Bot client connects without anticheat rejection
- **WHEN** a bot client connects to the bot server
- **THEN** the server accepts the connection regardless of packet rate or action frequency

### Requirement: XP multiplier is configurable via INI
The bot server SHALL read an `XpMultiplier` setting from `Config.ini` and apply it to all skill XP gains, so benchmark runs complete in less wall-clock time.

#### Scenario: XP multiplier applied
- **WHEN** `XpMultiplier=10` is set and a character gains XP
- **THEN** the XP awarded is 10× the base formula result

#### Scenario: Default multiplier behaves normally
- **WHEN** `XpMultiplier` is absent or set to 1
- **THEN** XP gain follows the base formula unchanged

### Requirement: Account creation requires no email validation
The bot server SHALL allow account creation and login without any email confirmation step, so bot characters can be provisioned programmatically.

#### Scenario: Account active immediately after creation
- **WHEN** an account is created via the standard login protocol on the bot server
- **THEN** the account is immediately usable without an email confirmation step
