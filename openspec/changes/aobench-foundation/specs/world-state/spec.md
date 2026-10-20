# Spec Delta

## Purpose

A structured JSON snapshot of the live game state for a bot character, assembled by `BotAPI.bas` from the ao-libre `UserList` and map data, returned in response to a `getState` command.

## ADDED Requirements

### Requirement: getState returns a complete WorldState snapshot
`{"action":"getState","charname":"..."}` SHALL return a JSON object covering all state an agent needs: position, HP/mana/stamina, gold, exp, level, class, heading, skills, inventory, nearby NPCs, nearby players, and active status effects.

#### Scenario: Full state returned for logged-in character
- **WHEN** a getState command is received for a character that is logged in
- **THEN** the response includes `pos`, `hp`, `mana`, `stamina`, `gold`, `exp`, `level`, `clase`, `heading`, `skills`, `inventory`, `nearbyNpcs`, `nearbyPlayers`, and `statusEffects`

#### Scenario: getState for unknown character returns error
- **WHEN** a getState command names a character not logged in
- **THEN** the response is `{"ok":false,"error":"character not found"}`

### Requirement: Position includes map, x, and y
The `pos` field SHALL contain `{"map":<int>,"x":<int>,"y":<int>}` matching the character's current `UserList(i).pos` values.

#### Scenario: Position matches in-game location
- **WHEN** a character is at map 1, tile (50, 60)
- **THEN** `state.pos` is `{"map":1,"x":50,"y":60}`

### Requirement: Vitals include current and max values
`hp`, `mana`, and `stamina` SHALL each be `{"current":<int>,"max":<int>}` drawn from `UserList(i).Stats.MinHp/MaxHp`, `MinMAN/MaxMAN`, and `MinSta/MaxSta`.

#### Scenario: HP values match server stats
- **WHEN** a character has 80 HP of 150 max
- **THEN** `state.hp` is `{"current":80,"max":150}`

### Requirement: Skills maps all 24 ao-libre skill names to their level
The `skills` field SHALL be a JSON object keyed by skill name (Magia, Robar, Tacticas, Armas, Defensa, Apuñalar, Ocultarse, Comerciar, Liderazgo, Proyectiles, Meditar, Supervivencia, Navegar, Equitación, Resistencia, Talar, Pescar, Minería, Herrería, Carpintería, Alquimia, Sastrería, Domar, Wrestling) with integer level values from `UserList(i).Stats.UserSkills`.

#### Scenario: Skill level returned by name
- **WHEN** a character has Magia level 45
- **THEN** `state.skills.Magia` is `45`

### Requirement: Inventory lists all occupied slots
The `inventory` field SHALL be an array of `{"slot":<1-20>,"itemIndex":<int>,"name":"<string>","amount":<int>,"equipped":<bool>}` for every non-empty inventory slot, with item name resolved from the DAT data loaded at server startup.

#### Scenario: Occupied slot appears in inventory
- **WHEN** slot 3 holds 5 iron swords
- **THEN** inventory contains `{"slot":3,"itemIndex":192,"name":"Espada de Hierro","amount":5,"equipped":false}`

### Requirement: nearbyNpcs lists all NPCs in visible range
The `nearbyNpcs` field SHALL be an array of `{"charIndex":<int>,"name":"<string>","type":"<npcType>","pos":{"x":<int>,"y":<int>},"healthStatus":"<Intact|LightlyWounded|Wounded|SeriouslyWounded|AlmostDead>","isFighting":<bool>}` for every NPC visible to the character.

#### Scenario: NPC in range appears in nearbyNpcs
- **WHEN** an NPC is within visible range
- **THEN** it appears in `nearbyNpcs` with its name, type, position, and health status

#### Scenario: No NPCs returns empty array
- **WHEN** no NPCs are in visible range
- **THEN** `nearbyNpcs` is `[]`

### Requirement: statusEffects lists all active negative conditions
The `statusEffects` field SHALL be an array of strings listing every active flag from `UserList(i).flags` that affects gameplay: `"dead"`, `"paralyzed"`, `"poisoned"`, `"blind"`, `"dumb"`, `"invisible"`, `"hidden"`, `"meditating"`, `"resting"`.

#### Scenario: Poisoned character shows status effect
- **WHEN** a character has the poisoned flag set
- **THEN** `statusEffects` contains `"poisoned"`

#### Scenario: Healthy character returns empty array
- **WHEN** no negative flags are set
- **THEN** `statusEffects` is `[]`
