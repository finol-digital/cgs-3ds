# cgs-3ds Plan

Status: draft, 2026-11-01. Owner: David Finol.

## Goal

Play Card Game Simulator (CGS) games on the Nintendo 3DS family, starting with
a New Nintendo 2DS XL dev/debug unit. Requirements:

1. The three built-in CGS games must work well: Standard Playing Cards, Mahjong,
   Dominoes (`Card-Game-Simulator/Assets/StreamingAssets/*@www.cardgamesimulator.com`).
2. Any CGS game (the `cgs.json` / `FinolDigital.Cgs.Json` ecosystem) should work,
   within hardware limits.
3. Cross-platform multiplayer with every CGS platform (desktop, mobile, web,
   XR) and 3DS-to-3DS.

## Constraints and decisions

| # | Decision | Reason |
|---|----------|--------|
| D1 | Native C/C++ homebrew with devkitPro (devkitARM, libctru, citro2d/citro3d) | No CTR-SDK access. Unity dropped 3DS after 2017.4, and CGS needs Unity 6. The 3DS has no .NET runtime and its browser has no WebAssembly. |
| D2 | Reuse CGS *data formats*, not code | `cgs.json`, `AllCards.json`/`AllSets.json`, decks, and image URL templates are the compatibility contract. |
| D3 | The PC-side converter lives here (`tools/`), in C# | It can reference `FinolDigital.Cgs.Json` so it parses games exactly as CGS does. Later it moves to the `FinolDigital.Cgs.Json` repo with the other shared logic (including from `Card-Game-Simulator-XR`). |
| D4 | Cross-play needs a new, open, engine-neutral protocol (working name **CGS Table Protocol, CTP**) | CGS multiplayer is Netcode for GameObjects (~300 RPC/NetworkVariable sites) over Unity Transport plus Unity Relay/Lobby. Rebuilding it in C, and authenticating with Unity Services from a 3DS, are both impractical. |
| D5 | Develop on an emulator (Azahar/Lime3DS/Citra-family) first, then on hardware | The debug unit is valuable and hard to replace. Hardware access must never block development. |
| D6 | MIT license (public repo) | Permissive and common for homebrew. MIT code can move into the MPL-2.0 shared repos later; MPL-2.0 CGS code must not be copied in here (dependencies such as `FinolDigital.Cgs.Json` via NuGet are fine). |

### Target hardware (New 2DS XL)

- CPU: ARM11 MPCore at 804 MHz (New-3DS clock and L2 cache via `osSetSpeedupEnable`).
- RAM: about 124 MB per app by default, up to about 178 MB with extended memory mode on New-3DS.
- GPU: PICA200. Textures must be power-of-two and at most 1024×1024. Use ETC1, RGB565 or RGBA4 to save memory.
- Screens: 400×240 top, 320×240 resistive touch bottom. No stereoscopic 3D on the 2DS.
- Network: 2.4 GHz 802.11b/g only, with WEP/WPA/WPA2 (no WPA3). The system TLS is outdated, so bundle mbedTLS.
- Local wireless (UDS) for 3DS-to-3DS play without an access point.

### Debug-unit specifics (verify in Phase 0)

The debug unit runs development-signed firmware. Before changing anything on it:

1. Find out its exact type (e.g. PARTNER-CTR Debugger vs. a CTR debug "panda" unit), firmware version, and region.
2. Find out how to run unsigned homebrew on a debug unit. Retail CFW guides
   (boot9strap/Luma3DS) assume retail keys. Debug units need dev-specific builds and
   steps, and retail-signed `.cia` files will not install. Prefer the route
   that changes the least, e.g. `.3dsx` through a Homebrew Launcher.
3. Back up NAND and OTP before any firmware-level change.
4. Until this is solved, all work continues on the emulator.

## Architecture

```
cgs-3ds/
  app/            3DS client (C/C++, devkitARM, citro2d) -> .3dsx (+ .cia later)
  tools/converter C# (.NET) CLI: CGS game -> 3DS game pack
  protocol/       CTP spec + test vectors (later moves to FinolDigital.Cgs.Json)
  docs/
```

### 3DS game pack (converter output)

Big games (thousands of cards and large JSON files) cannot be parsed or held in
memory on the 3DS, so the converter does the heavy work:

- `pack.json`: game metadata from `cgs.json` that the client needs (card size,
  play mat, zones, deck formats, properties to show, game rules for counters/dice, and so on).
- `cards.idx` plus `cards.dat`: binary card index sorted for search by name and
  set, loaded in pages instead of all at once.
- `img/<id>.t3x`: one texture per card with a thumbnail (about 64×89) and a zoom size (about 256×358). Card backs, play mat and banner are converted the same way.
- `decks/`: the game's built-in decks, unchanged from CGS.
- A version and source hash so packs can be regenerated and cached.

Packs are delivered by SD card copy, or downloaded on the device from a static
host (GitHub Pages or a Cloudflare R2 bucket next to `www.cardgamesimulator.com`).
Long term, the converter could run in CI for every game in the CGS games
catalog, so any game would have a ready-made pack.

### 3DS client

- Rendering with citro2d. Top screen: zoomed card, table overview, scoreboard.
  Bottom screen: hand, selected zone, menus (touch plus D-pad/Circle Pad).
- Image cache: a fixed-size LRU of GPU textures, filled in the background on a
  worker thread from SD (`img/*.t3x`).
- Game logic modeled on CGS's table model: board/play mat, card zones,
  stacks, cards (face up/down, rotation, owner), counters, dice.
- Decks: import and export CGS deck text formats.
- Networking: CTP client and host (see below), over TCP/WebSocket on
  infrastructure Wi-Fi and over UDS for local play.

## Cross-platform multiplayer: CGS Table Protocol (CTP)

CTP is a small, documented, host-authoritative protocol for a shared table.
It is written down as a spec with test vectors so C (3DS), C# (Unity CGS, XR)
and JavaScript (web) implementations stay compatible.

- **Model:** the host holds the authoritative table state (game id and version,
  players, stacks/zones, cards with id, position, rotation, face and owner,
  counters, dice). Clients send intents (move, flip, shuffle, draw, roll, and so on).
  The host validates them and broadcasts state changes. A full snapshot is sent on join and resync.
- **Encoding:** binary (e.g. CBOR or a custom fixed layout) with a version
  handshake. Card ids are CGS card ids, so every client resolves images on its own.
- **Transports:**
  - LAN: TCP plus UDP broadcast discovery (works on 3DS, desktop and mobile).
  - Internet: a **WebSocket relay**, with rooms as Cloudflare Durable Objects
    (CGS already has `edge-worker/` on Cloudflare). WebSockets work from
    browsers, Unity and the 3DS (mbedTLS plus a small WS client). No Unity
    Services authentication needed.
  - 3DS local wireless: CTP frames over UDS (3DS-to-3DS only).
- **Adoption in Unity CGS** (this needs work in the CGS repo):
  1. *Bridge (first):* a Unity CGS host also speaks CTP and maps it to its
     Netcode state, so 3DS players can join Unity-hosted rooms.
  2. *Native (later):* CGS can join CTP rooms, including 3DS-hosted and
     relay-hosted ones. CTP may eventually replace NGO in CGS and XR, which also
     removes the Unity Relay/Lobby dependency.
- **Mixed tables:** each client must have the game installed (on 3DS, the pack).
  The handshake checks the game id and version.

## Phases

| Phase | Deliverable | Exit criteria |
|-------|-------------|---------------|
| 0. Foundations | devkitPro toolchain, `app/` skeleton (Makefile, CI build of `.3dsx`), emulator workflow; debug-unit homebrew research and NAND backup | Hello-world runs in the emulator and on the 2DS; documented, reversible debug-unit setup |
| 1. Converter | `tools/converter` for the 3 built-in games, with unit tests against CGS's own copies | Packs for Standard Playing Cards, Mahjong, Dominoes; images are exact or close matches |
| 2. Viewer | Load a pack, browse/search cards, zoomed view on top screen | 60 fps scrolling; memory stays within a fixed budget for all 3 games |
| 3. Solo table | Play mat, zones, stacks, drag/flip/rotate, shuffle/draw, counters, dice, built-in decks | One full solo game of each built-in game playable |
| 4. CTP spec | `protocol/` spec, test vectors, C reference library; LAN host/join between 3DS units (emulator plus hardware) | Two 3DS units play Standard Playing Cards over LAN and over UDS |
| 5. Unity bridge | CGS repo: CTP host bridge (separate OpenSpec change there) | A 3DS joins a desktop-hosted CGS LAN room and both sides see the same state |
| 6. Internet relay | Cloudflare WebSocket relay; 3DS and CGS support | 3DS ↔ desktop/mobile/web play over the Internet |
| 7. Ecosystem | Converter for all games, CI-generated pack catalog, on-device download, deck import | Most catalog games playable; documented limits for very large games |
| 8. Consolidation | Move converter, CTP spec and shared logic to `FinolDigital.Cgs.Json`; CGS and XR adopt native CTP | One shared implementation per language |

## Open questions

- Exact debug-unit model and firmware, and the safest homebrew route (Phase 0).
- CTP encoding (CBOR vs. custom binary) and how much game-rule logic runs on
  the host vs. on each client.
- Whether CTP should replace NGO in CGS/XR or exist next to it permanently.
- Hosting and cost of the relay and of the pack catalog.
- Size limits for very large games (e.g. Magic). Possible answers: partial packs by set, or downloading images on demand.
