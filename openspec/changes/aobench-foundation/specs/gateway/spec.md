# Spec Delta

## Purpose

A Bun WebSocket process (`packages/gateway`) that sits between `BotAPI.bas` and the SDK, routing state snapshots and actions, managing `control`/`observe` session modes, and exposing HTTP endpoints so tools can query bot state without a persistent WebSocket connection.

## ADDED Requirements

### Requirement: Gateway connects to BotAPI.bas as a client
The gateway SHALL maintain a persistent TCP JSON connection to `BotAPI.bas` on port 4200, poll `getState` on each connected bot at every game tick (~600ms), and relay results to subscribed SDK sessions.

#### Scenario: Gateway receives state from BotAPI.bas
- **WHEN** the gateway polls `getState` for a connected bot
- **THEN** it stores the latest `WorldState` and fans it out to all subscribed SDK sessions

#### Scenario: Gateway reconnects after BotAPI.bas restart
- **WHEN** the TCP connection to BotAPI.bas drops
- **THEN** the gateway retries with exponential backoff and resumes polling once reconnected

### Requirement: SDK connects to gateway via WebSocket
The SDK SHALL connect to the gateway via WebSocket (default port 7780) by sending `{"type":"sdk_connect","username":"<bot>","password":"<pass>","mode":"control"|"observe"}`. The gateway authenticates and routes all subsequent actions and state frames.

#### Scenario: SDK connects in control mode
- **WHEN** an SDK sends `sdk_connect` with `mode: "control"`
- **THEN** the gateway registers it as the controller and sends the current WorldState immediately

#### Scenario: SDK connects in observe mode
- **WHEN** an SDK sends `sdk_connect` with `mode: "observe"`
- **THEN** the gateway registers it as an observer; it receives state but actions (except `say`) are rejected

### Requirement: Last controller wins
When a new SDK connects in `control` mode while another controller is already connected, the gateway SHALL disconnect the old controller with an error message and register the new one.

#### Scenario: New controller pre-empts old one
- **WHEN** a second `sdk_connect` with `mode: "control"` arrives for the same bot
- **THEN** the first controller receives `{"type":"sdk_error","error":"Disconnected: another controller connected"}` and is closed

### Requirement: Multiple observers may coexist
Any number of `observe`-mode SDK sessions SHALL be able to connect to the same bot simultaneously. Each receives all state frames.

#### Scenario: Two observers connected simultaneously
- **WHEN** two SDK sessions connect with `mode: "observe"` for the same bot
- **THEN** both receive state updates and neither is disconnected

### Requirement: Gateway deduplicates identical state frames
The gateway SHALL NOT fan out a state frame to SDK subscribers if it is identical to the last frame sent (same tick, same content), saving bandwidth.

#### Scenario: Duplicate frame suppressed
- **WHEN** BotAPI.bas returns the same WorldState on two consecutive polls
- **THEN** the gateway sends only one frame to subscribers

### Requirement: HTTP GET /state/:bot returns latest WorldState
`GET /state/:bot` SHALL return `{"state":<WorldState>,"stateAge":<ms>}` if the bot is connected, or HTTP 404 if not.

#### Scenario: State available via HTTP
- **WHEN** `GET /state/bot1` is called while bot1 is connected
- **THEN** the response contains the latest WorldState and the age in milliseconds

### Requirement: HTTP GET /chat/:bot returns recent chat
`GET /chat/:bot?limit=20` SHALL return the last N chat messages accumulated from state frames for the named bot.

#### Scenario: Chat history returned via HTTP
- **WHEN** `GET /chat/bot1?limit=10` is called
- **THEN** the response contains the 10 most recent chat messages seen by bot1

### Requirement: HTTP POST /say/:bot sends a chat message
`POST /say/:bot` with body `{"text":"..."}` SHALL relay the message through the bot character and return the result.

#### Scenario: Chat sent via HTTP
- **WHEN** `POST /say/bot1` is called with `{"text":"hola"}`
- **THEN** the bot character says "hola" in game and the response confirms success

### Requirement: Gateway exposes a status endpoint
`GET /status` SHALL return a JSON summary of all connected bots and SDK sessions with their modes and connection state.

#### Scenario: Status shows connected sessions
- **WHEN** `GET /status` is called
- **THEN** the response lists each bot with its connection status, tick, and SDK session count
