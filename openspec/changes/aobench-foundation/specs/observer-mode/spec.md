# Spec Delta

## Purpose

A second client connection that attaches to a running bot character and receives all server→client packets in real time, allowing a VB6 client to watch the bot play — seeing its movement, combat, HP changes, chat, inventory, and spell effects — without being able to send action packets that would interfere with the bot.

## ADDED Requirements

### Requirement: Server accepts a new LoginObserver packet
The ao-libre server fork SHALL handle a new `LoginObserver` client packet (appended to `ClientPacketID`) that includes a target charname and an observer password. The server SHALL find the named character's `UserIndex`, store the observer's connection ID on `UserList(i).ObserverConnectionID`, and replay the character's current state to the observer (map load, position, HP, inventory, skills).

#### Scenario: Observer attaches to a running bot
- **WHEN** a client sends `LoginObserver` with the bot's charname and correct observer password
- **THEN** the server stores the observer connection on that `UserIndex` and sends the bot's current map, position, and stats to the observer

#### Scenario: Observer attach rejected for unknown character
- **WHEN** `LoginObserver` names a character not currently logged in
- **THEN** the server sends an error packet and closes the observer connection

#### Scenario: Observer attach rejected for wrong password
- **WHEN** `LoginObserver` provides an incorrect observer password
- **THEN** the server sends an error packet and closes the observer connection

### Requirement: All server→client packets sent to the bot are mirrored to the observer
Every packet the server sends to `UserIndex` via `SendData(ToUser, UserIndex, ...)` SHALL also be written to `UserList(i).ObserverConnectionID` if it is set, so the observer receives an identical stream.

#### Scenario: Bot moves — observer sees it
- **WHEN** the bot character moves and the server sends `PosUpdate` to the bot
- **THEN** the observer receives the same `PosUpdate` packet

#### Scenario: Bot takes damage — observer sees HP change
- **WHEN** the bot takes damage and the server sends `UpdateHP` to the bot
- **THEN** the observer receives the same `UpdateHP` packet

#### Scenario: NPC appears — observer sees it
- **WHEN** a nearby NPC triggers `CharacterCreate` to the bot's area
- **THEN** the observer receives the same `CharacterCreate` packet

### Requirement: Action packets from the observer are silently dropped
Any `ClientPacketID` packet arriving from the observer connection SHALL be silently discarded, with the exception of `Talk` which the observer MAY send to be relayed as console-only text to the bot character.

#### Scenario: Observer sends Walk — ignored
- **WHEN** the observer connection sends a `Walk` packet
- **THEN** the server discards it and the bot character does not move

#### Scenario: Observer sends Talk — relayed to bot's console
- **WHEN** the observer connection sends a `Talk` packet
- **THEN** the message is appended to the bot character's chat console only (not broadcast to the world)

### Requirement: Observer disconnection does not affect the bot
When the observer connection drops, the server SHALL clear `UserList(i).ObserverConnectionID` and continue running the bot character normally.

#### Scenario: Observer disconnects — bot continues
- **WHEN** the observer's TCP connection closes
- **THEN** the bot character remains logged in and the bot session continues unaffected

### Requirement: Only one observer per bot character at a time
The server SHALL allow at most one observer connection per bot character. If a second observer tries to attach while one is already connected, the server SHALL reject it.

#### Scenario: Second observer rejected
- **WHEN** a second `LoginObserver` for the same charname arrives while an observer is connected
- **THEN** the server sends an error packet to the second connection and closes it

### Requirement: ao-libre client fork supports observer login mode
The ao-libre client fork SHALL add an observer login screen that accepts a server address, bot charname, and observer password, sends `LoginObserver`, and then renders the game normally in read-only mode — all action inputs (keyboard movement, clicks) are disabled, only the camera follows the bot character's position.

#### Scenario: Observer client renders bot's view
- **WHEN** the observer client connects and attaches to a bot
- **THEN** the client renders the bot's map, position, surrounding entities, HP bar, chat, and spell effects in real time

#### Scenario: Action inputs disabled in observer mode
- **WHEN** the observer presses the movement keys or clicks to attack
- **THEN** no action packet is sent and the client shows a visual indicator that it is in observe-only mode
