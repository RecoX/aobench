# Design

## Context

ao-libre (`ao-libre/ao-server`, `ao-libre/ao-cliente`) is the original open-source Argentum Online codebase (AGPL). It uses standard Winsock via `MSWINSCK.OCX` for game networking, a well-documented binary protocol with a clean `ClientPacketID`/`ServerPacketID` enum, and already has an `ApiConnection.bas` module (written by Recox) that makes outbound HTTP calls using `Microsoft.XmlHttp`. The server runs on Windows, compiles with VB6, and stores character data in flat files under `Charfile/`. The game port defaults to 7666.

The project is a new monorepo `recox/aobench` with a companion fork `recox/ao-libre-server` (ao-libre + `BotAPI.bas`).

## Goals / Non-Goals

**Goals:**
- Agents control characters via a clean TypeScript SDK, no knowledge of the binary game protocol required
- All bot traffic goes through a dedicated ao-libre server instance isolated from production
- `BotAPI.bas` is a minimal addition to ao-libre — it calls into existing handler logic, not duplicates it
- The SDK mirrors rs-sdk's patterns so it is familiar to anyone who has used runebench
- The leaderboard and harness are deployable independently of the bot server

**Non-Goals:**
- Implementing the binary AO game protocol in TypeScript (BotAPI.bas sidesteps this entirely)
- Modifying the production AO20 server in any way
- Supporting real player connections through the BotAPI port
- Real-time event streaming in v1 (polling via getState is sufficient)
- Forking ao-libre/ao-cliente for runtime use (reference only)

## Decisions

### Decision 1: BotAPI.bas connects outbound to the gateway; SDK connects to gateway via WebSocket

**Chosen:** `BotAPI.bas` maintains a TCP JSON connection to the gateway process (Bun, port 7780). The SDK connects to the gateway via WebSocket. The gateway fans state out to SDK sessions, routes actions to BotAPI.bas, and exposes HTTP endpoints. This mirrors rs-sdk's architecture exactly.

**Alternative considered:** SDK connects directly to BotAPI.bas TCP port 4200 (original plan).

**Rationale:** The direct TCP approach works but is monolithic — every SDK client manages its own TCP connection to VB6, there's no session management, no control/observe mode separation, no HTTP fallback for tools. The gateway pattern decouples the VB6 server from the TypeScript ecosystem, lets multiple SDK sessions observe the same bot, and provides HTTP endpoints (`GET /state`, `GET /chat`, `POST /say`) that the MCP server and CLI tools use without WebSocket connections.

---

### Decision 2: BotAPI.bas calls into existing ao-libre protocol handler logic

**Chosen:** `BotAPI.bas` parses the incoming JSON, extracts parameters, and calls the same VB6 subroutines that the binary protocol handlers call (e.g. the same movement, attack, skill logic) rather than duplicating game logic.

**Alternative considered:** Execute the binary protocol path end-to-end internally.

**Rationale:** The binary protocol path includes packet parsing, timestamp verification, and anti-macro checks that are irrelevant for controlled bot execution. Calling the underlying game logic directly (as AO20's `BotCommands.bas` pattern would) is cleaner and less likely to break with future ao-libre changes.

---

### Decision 3: WorldState is assembled from UserList on demand (pull/polling)

**Chosen:** `getState` reads `UserList(i)` fields at the moment of the request and serializes them to JSON. The SDK polls as needed.

**Alternative considered:** Push events to the SDK when server state changes (OnHP, OnMove, etc.).

**Rationale:** Push requires the server to emit structured JSON for every game event, a much larger VB6 change. Polling is sufficient for agent decision-making — agents are not latency-sensitive at the millisecond level. Can be upgraded to push in a future change.

---

### Decision 4: Character lookup uses a name → UserIndex map in BotAPI.bas

**Chosen:** `BotAPI.bas` maintains a `Dictionary` mapping `charname → UserIndex` updated on bot login/logout. Commands look up the index from this map.

**Rationale:** ao-libre's `UserList` is indexed by integer slot, not name. A Dictionary avoids O(N) scans on every command. The same pattern is used elsewhere in ao-libre.

---

### Decision 5: Agent code sandbox uses a Bun subprocess with an import allowlist

**Chosen:** The harness spawns a Bun subprocess per agent turn. A custom import hook blocks everything except `@aobench/sdk` and Bun builtins. The subprocess communicates results via stdout; a timeout kills it.

**Alternative considered:** vm2, Node's `vm` module, Deno.

**Rationale:** `vm2` is unmaintained with known escapes. Node `vm` doesn't isolate `import`. Deno adds a large runtime dependency. Bun's subprocess model with a module allowlist is robust and matches the runtime we use for the SDK itself.

---

### Decision 6: Monorepo with Bun workspaces

**Chosen:** `recox/aobench` is a Bun workspace monorepo: `packages/sdk`, `packages/harness`, `packages/leaderboard`, `packages/wiki-extractor`.

**Rationale:** All four packages share types and task definitions. A monorepo simplifies cross-package imports during early development. The SDK and leaderboard can be published separately once stable.

---

### Decision 7: Leaderboard uses Nuxt 4 + SQLite

**Chosen:** Nuxt 4 SPA with SQLite via better-sqlite3 in Nitro server routes.

**Rationale:** Matches ao20-web conventions (Tailwind 4 + DaisyUI 5). SQLite is sufficient for leaderboard volumes and avoids an external database dependency in v1.

### Decision 8: ao-libre has 20 skills and 4 headings — SDK reflects this exactly

**From deep analysis of ao-libre `Declares.bas`:** The `eSkill` enum has 20 values (Magia, Robar, Tacticas, Armas, Meditar, Apuñalar, Ocultarse, Supervivencia, Talar, Comerciar, Defensa, Pesca, Minería, Carpintería, Herrería, Liderazgo, Domar, Proyectiles, Wrestling, Navegación). `E_Heading` has only 4 directions (N/S/E/W). The wiki-extractor and SDK must reflect this — not the AO20 values (24 skills, 8 headings) referenced in earlier drafts.

Also from deep analysis: `JSON.bas` already exists in ao-libre client's `CODIGO/IO/` folder — it can be copied directly into the server fork for VBJson support, no third-party module needed.

**Videos/ folder:** ao-libre client has an empty `Videos/` folder (`.gitkeep` placeholder) with DirectShow infrastructure (`IMediaControl` in `clsAudio.cls`) planned for video playback. FFmpeg is **not present** in either ao-libre repo. For benchmark run recording, OBS or any screen capture tool on the Windows observer VM is sufficient — no code changes needed.

## Risks / Trade-offs

**VB6 changes are the critical path**
→ Everything depends on `BotAPI.bas` working. Build a mock BotAPI server in Node.js (same TCP JSON protocol) early so the SDK and harness can be developed and tested without a running VB6 server.

**ao-libre character file format must be understood to provision/delete bot accounts**
→ The harness needs to create and delete charfiles programmatically. The format is in the ao-libre `Charfile/` folder — plain text INI-style files. Read and document this before implementing the harness provisioning step.

**JSON parsing in VB6 requires a library**
→ ao-libre does not currently include a JSON parser. Must add a VB6 JSON library (e.g. the `VBJson` module already present in ao-libre/ao-cliente, or a minimal custom parser). Keep it dependency-free — copy the `.bas` file directly into the fork.

**Winsock control array limit**
→ VB6 Winsock control arrays have a practical limit of ~200 instances. For v1 (single benchmark runs) this is fine. Document the limit.

**ao-libre server runs on Windows**
→ The bot server VM must be Windows. This is expected — ao-libre is a VB6 application.

## Migration Plan

1. Fork `ao-libre/ao-server` → `recox/ao-libre-server`; verify it compiles and starts
2. Copy `JSON.bas` from ao-libre client `CODIGO/IO/` into the server fork; verify JSON parse/serialize works
3. Add `frmBotAPI.frm` + `BotAPI.bas` with only `getState` implemented; test with `nc` on port 4200
4. Implement remaining commands in `BotAPI.bas` one at a time, verifying each against a live character
5. Build `packages/gateway` — connect to BotAPI.bas, verify state fan-out via WebSocket
6. Build and publish `@aobench/sdk` connecting via gateway WebSocket
7. Build wiki-extractor against ao-libre DAT files (20 skills, 4 headings); generate first wiki docs
8. Build harness with a scripted (non-LLM) test agent; verify end-to-end XP measurement
9. Build MCP server connecting via gateway
10. Build bot-cli tools (create-bot, cli, bug-report)
11. Implement observer mode (LoginObserver + packet mirroring + read-only client fork)
12. Deploy leaderboard; submit a test run
13. Run first real LLM benchmark

## Open Questions

- **XP multiplier value:** What multiplier makes a 15-minute run cover meaningful skill progression? Needs empirical testing.
- **Gateway poll rate:** Should the gateway poll BotAPI.bas at every game tick (600ms) or faster? Too fast and we add VB6 load; too slow and state feels stale for agents. Start at 600ms (1 tick), adjust based on testing.
