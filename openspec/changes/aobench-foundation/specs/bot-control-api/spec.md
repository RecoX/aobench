# Spec Delta

## Purpose

A new VB6 module `BotAPI.bas` in the ao-libre server fork that opens a TCP JSON listener on port 4200, accepts bot command messages, executes them by calling into existing ao-libre protocol handler logic, and returns JSON responses — all without requiring callers to speak the binary game protocol.

## ADDED Requirements

### Requirement: BotAPI.bas opens a TCP JSON listener on port 4200
`BotAPI.bas` SHALL use a Winsock control array to listen on port 4200, accept multiple simultaneous bot connections, read newline-delimited JSON messages, and dispatch each to the appropriate handler.

#### Scenario: Listener accepts a connection
- **WHEN** a TCP client connects to port 4200
- **THEN** the server accepts it and waits for a JSON command message

#### Scenario: Multiple bots connect simultaneously
- **WHEN** N bot clients connect concurrently
- **THEN** each gets its own socket slot and commands are dispatched independently

### Requirement: Each command message identifies the target character
Every bot command JSON message SHALL include a `charname` field identifying which connected player character to act on, so one BotAPI listener serves all active bot sessions.

#### Scenario: Command routed to correct character
- **WHEN** `{"action":"walk","charname":"bot1","heading":"north"}` is received
- **THEN** only the character named `bot1` walks north

#### Scenario: Unknown character returns error
- **WHEN** a command names a character not currently logged in
- **THEN** the response is `{"ok":false,"error":"character not found"}`

### Requirement: walk command moves a character one step
`{"action":"walk","charname":"...","heading":"<N|S|E|W|NE|NW|SE|SW>"}` SHALL move the character one tile in the given heading by calling the same logic as `HandleWalk`.

#### Scenario: Valid walk executed
- **WHEN** a walk command with a valid heading is received
- **THEN** the character moves one tile and the response is `{"ok":true}`

### Requirement: attack command attacks the current target
`{"action":"attack","charname":"..."}` SHALL trigger a melee attack on the character's current target (set by leftClick) using the same logic as `HandleAttack`.

#### Scenario: Attack executed with valid target
- **WHEN** an attack command is received and the character has a target
- **THEN** the attack is processed and the response is `{"ok":true}`

### Requirement: leftClick command sets the target
`{"action":"leftClick","charname":"...","x":<dx>,"y":<dy>}` SHALL target the entity or tile at the given offset from the character, using `HandleLeftClick` logic.

#### Scenario: NPC targeted via leftClick
- **WHEN** a leftClick command is received pointing at a tile with an NPC
- **THEN** that NPC becomes the character's target

### Requirement: castSpell command casts a spell slot
`{"action":"castSpell","charname":"...","slot":<1-35>}` SHALL cast the spell in the given inventory slot using `HandleCastSpell` logic.

#### Scenario: Spell cast from valid slot
- **WHEN** a castSpell command is received with a valid slot
- **THEN** the spell is cast and the response is `{"ok":true}`

### Requirement: useItem command uses an inventory item
`{"action":"useItem","charname":"...","slot":<1-20>}` SHALL use the item in the given inventory slot using `HandleUseItem` logic.

#### Scenario: Item used from valid slot
- **WHEN** a useItem command is received with a valid slot containing an item
- **THEN** the item effect is applied and the response is `{"ok":true}`

### Requirement: work command uses a skill
`{"action":"work","charname":"...","skill":"<skillName>"}` SHALL activate the named skill (Magia, Herrería, Pesca, Talar, Minería, etc.) using `HandleWork` logic.

#### Scenario: Skill activated
- **WHEN** a work command is received with a valid skill name
- **THEN** the skill action is triggered and the response is `{"ok":true}`

### Requirement: talk command sends a chat message
`{"action":"talk","charname":"...","message":"<text>"}` SHALL send the message to nearby chat using `HandleTalk` logic.

#### Scenario: Chat message sent
- **WHEN** a talk command is received with a non-empty message
- **THEN** the message appears in the game world and the response is `{"ok":true}`

### Requirement: pickUp command picks up an item from the ground
`{"action":"pickUp","charname":"...","x":<dx>,"y":<dy>}` SHALL pick up the item at the given tile offset using `HandlePickUp` logic.

#### Scenario: Item picked up
- **WHEN** a pickUp command is received pointing at a tile with an item
- **THEN** the item moves to the character's inventory and the response is `{"ok":true}`

### Requirement: equipItem command equips an inventory slot
`{"action":"equipItem","charname":"...","slot":<1-20>}` SHALL equip the item in the given slot using `HandleEquipItem` logic.

#### Scenario: Item equipped
- **WHEN** an equipItem command is received with a valid equippable slot
- **THEN** the item is equipped and the response is `{"ok":true}`

### Requirement: meditate command toggles meditation
`{"action":"meditate","charname":"..."}` SHALL toggle meditation on the character using `HandleMeditate` logic.

#### Scenario: Meditation toggled
- **WHEN** a meditate command is received
- **THEN** the character begins or ends meditation and the response is `{"ok":true}`

### Requirement: rest command toggles rest
`{"action":"rest","charname":"..."}` SHALL toggle rest on the character using `HandleRest` logic.

#### Scenario: Rest toggled
- **WHEN** a rest command is received
- **THEN** the character begins or ends resting and the response is `{"ok":true}`

### Requirement: drop command drops an item
`{"action":"drop","charname":"...","slot":<1-20>,"amount":<n>}` SHALL drop the given quantity of the item in the slot onto the ground.

#### Scenario: Item dropped
- **WHEN** a drop command is received with a valid slot and amount
- **THEN** the item appears on the ground and the response is `{"ok":true}`

### Requirement: Invalid or malformed commands return a structured error
Any command with a missing required field, an invalid action name, or a value out of range SHALL return `{"ok":false,"error":"<reason>"}` without crashing the server.

#### Scenario: Unknown action returns error
- **WHEN** a command with `"action":"fly"` is received
- **THEN** the response is `{"ok":false,"error":"unknown action"}` and the server continues running
