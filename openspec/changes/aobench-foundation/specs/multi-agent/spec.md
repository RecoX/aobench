# Spec Delta

## Purpose

Multi-agent benchmark tasks where two or more bot characters interact with each other in the same game world — forming parties, trading items, coordinating combat, or competing as enemies — enabling evaluation of LLM coordination and communication beyond single-player optimization.

> **Status: v2 — out of scope for v1 implementation. The gateway and SDK already support multiple simultaneous bot sessions. This spec defines what needs to be added on top.**

## ADDED Requirements

### Requirement: Harness supports multi-bot task definitions
A task YAML MAY specify `bots: N` to provision N bot characters simultaneously. Each bot gets its own SDK instance connected via the gateway. The harness passes all N SDK instances to the agent as `bot1`, `bot2`, etc.

#### Scenario: Two-bot task provisioned
- **WHEN** a task definition specifies `bots: 2` and the harness starts
- **THEN** two bot characters are created, both logged in, and the agent receives two `AOBot` instances

### Requirement: Bots can form a party
`await bot1.invite(bot2.charname)` followed by `await bot2.joinParty()` SHALL form an in-game party using the `PartyCreate`/`PartyJoin` protocol, enabling party chat and shared XP.

#### Scenario: Two bots form a party
- **WHEN** bot1 invites bot2 and bot2 accepts
- **THEN** both characters are in the same party and can use `bot.partyMessage(msg)`

### Requirement: Bots can trade items directly
`await bot1.offerTrade(bot2.charname)` SHALL initiate a player commerce session between two bots using the `UserCommerceOffer`/`UserCommerceOk` protocol.

#### Scenario: Bots exchange an item
- **WHEN** bot1 initiates trade, offers an item, and bot2 accepts
- **THEN** the item moves from bot1's inventory to bot2's inventory

### Requirement: Bots can whisper to coordinate
`await bot.whisper(targetCharname, message)` SHALL send a private message visible only to the target bot via the existing `Whisper` protocol.

#### Scenario: Bot1 coordinates with bot2 via whisper
- **WHEN** bot1 sends `whisper(bot2.charname, "attack the lobo at 50,60")`
- **THEN** the message appears in bot2's incoming chat and bot2 can act on it

### Requirement: Multi-agent scoring measures combined output
For multi-bot tasks, the score SHALL measure the combined result — total XP across all bots per 15-second window, or a task-specific metric like time-to-kill a boss — rather than per-bot individual scores.

#### Scenario: Combined XP rate scored
- **WHEN** a two-bot task ends
- **THEN** the score is `max(bot1_xp_delta + bot2_xp_delta)` over any 15-second window

### Requirement: Observer mode supports watching multiple bots
For multi-bot runs, the observer client SHALL be able to switch between bots using a keyboard shortcut, or open multiple observer windows simultaneously.

#### Scenario: Observer switches between bots
- **WHEN** the observer presses Tab during a two-bot run
- **THEN** the view switches from bot1's perspective to bot2's perspective
