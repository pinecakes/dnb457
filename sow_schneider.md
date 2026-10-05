# CS 457 Project Statement of Work (SOW) & Protocol Specification Template

**Student Name:** Pine Schneider  
**Date:** 2026-09-19 
**Course:** CS 457 - Computer Networks  
**Target Server Domain:** `server.schneider.edu`  

---

## 1. Game Selection & Scope (Sprint 0)

> Planning is going to be an iterative process through the sprints so you don't have to have all the details now. Focus on big overview concepts. You will be updating the SOW as we plan.
> You have a lot of freedom to choose a game. There are a couple caveats.  

> - It must run in the console. The lab nodes won't be able to handle extensive graphics.
> - It has to be self-contained. You can use a internet-connector to download you code, but because the architecture must run 5 nodes you won't be able to run 
> - You are encouraged to use python, but I'm not going to make it a strict requirement. The instructor and TA's ability to help with C or Rust, etc will be diminished in other languages.

### 1.1 Game Overview
- **Chosen Game:** Dots and Boxes
- **Player Capacity:** 2 Players (Simulated via 2 CML Client nodes)
- **Game Summary:** Players will take turns designating which 2 points of a grid of dots they would like to draw a line between. If drawing a line will fully enclose a square then that square is claimed and counts as a point in that player's favor. The game ends when all squares have been claimed.

### 1.2 Core Game Rules & Win/Draw Conditions
- **Turn Mechanics:** The server will keep an ongoing Turn counter. When said counter is odd, only Player 1 may submit orders, Player 2 may not, and vice versa for even. It will increment upon a valid order.
- **Victory Condition:** Once all squares are claimed, the player with the most points wins.
- **Draw/Tie Condition:** Once all squares are claimed, if both players own 10 squares, they will tie.

---

## 2. Application-Layer Messaging Protocol Blueprint (Sprint 1 Deliverable)

### 2.1 Message Transport & Serialization Format
- **Transport Protocol:** TCP
- **Serialization Format:** JSON
- **Framing Mechanism:** Newline-delimited (`\n`) JSON payloads

### 2.2 Message Schema Definitions

#### Message Types:
1. `CONNECT` (Client -> Server): Request to join the game room.
2. `LOBBY_WAIT` (Server -> Client): Notification that server is waiting for Player 2.
3. `BEGIN` (Server -> Clients): Game initiated, empty board initialized, Turn 1.
4. `PLAYER_TURN` (Client -> Server): Player submission of coordinate pair.
5. `UPDATE_BOARD` (Server -> Clients): Broadcast current game board and Turn #.
6. `QUERY_BOARD` (Client -> Server): Client requests current board state and Turn #.
7. `GAME_OVER` (Server -> Clients): Victory / Draw / Forfeit notification with final scores.
8. `ERROR` (Server -> Client): Submission would result in invalid board or packet was malformed.

#### Example JSON Protocol Schema:
```json
{
  "msg_type": "PLAYER_TURN",
  "player_id": 1,
  "payload": {
    "pair1": [2, 4],
    "pair2": [2, 3]
  },
  "timestamp": 1727000000
}
```

---

### 2.3 Game State Machine (FSM) Design (Sprint 1 Deliverable)
```mermaid
---
config:
  look: classic
  layout: dagre
---
flowchart LR
    A["INIT"] -- start listening --> B["HOST_LOBBY"]
    B -- 2 players join --> C["BEGIN"]
    C -- assign roles and create empty board --> D["PLAYER_TURN"]
    D -- current player submits coordinate pair --> E["NEXT_BOARD"]
    E -- board is valid, advance to next turn --> H["UPDATE_BOARD"]
    H -- broadcast new board and turn to players --> D
    E -- board is invalid or not player's turn, return ERROR --> D
    E -- board is valid and all boxes claimed --> F["GAME_OVER"]
    D -- only 1 player is connected --> F
    F -- broadcast final results --> G["RESTART"]
    G -- Reset board and roles --> B
```

---

## 3. Game Behavior & Server Concurrency Architecture (Sprint 2 Deliverable)

### 3.1 Server Concurrency Strategy
- **Architecture Choice:** [Multi-Threading (`threading.Thread`) OR Non-blocking I/O multiplexing (`select.select` / `selectors`)]
- **Synchronization Logic:** Explain how shared game state and client list are thread-safe (e.g. `threading.Lock`) to prevent race conditions during turn processing.

### 3.2 State & Score Synchronization Across Clients
- **Turn Enforcement:** Detail how the server validates active player ID before processing moves and broadcasts updated turn notifications to all clients.
- **Score & Board Synchronization:** Describe how state broadcasts keep client screens synchronized in real time.

---

## 4. Coding & AI Implementation Plan (Sprint 3)

- **Permitted AI Tools:** [e.g., GitHub Copilot, ChatGPT, Claude]
- **AI Prompting & Constraint Strategy:** Explain how you will constrain AI models to generate code (in Python or your chosen language) that adheres strictly to the protocol blueprint and FSM designed in Sprints 1 & 2.
- **Implementation Risk Management:** Detail your plan to leverage past programming experience and manage time to ensure code completion on schedule.

---

## 5. CML Multi-Subnet Topology & Wireshark Deployment Plan (Sprint 4 & 5 Deliverable)

> For now you can use the topology below. We may update this when we get to defining subnets.

### 5.1 Subnet & Router Design
- **Subnet A (Client 1):** `192.168.10.0/24` (Interface `Gi0/1` on Router R1)
- **Subnet B (Client 2):** `192.168.11.0/24` (Interface `Gi0/2` on Router R1)
- **Subnet C (Game Server):** `192.168.20.0/24` (Interface `Gi0/1` on Router R2)
- **Router Backbone:** `10.0.0.0/30` (Interface `Gi0/0` on R1 <-> `Gi0/0` on R2)

### 5.2 DHCP Pools & DNS Configuration Plan
- **Router R1 DHCP Pool 1 (`CLIENT1_POOL`):** Leases `192.168.10.10` - `192.168.10.50`, gateway `192.168.10.1`, DNS `10.0.0.2`.
- **Router R1 DHCP Pool 2 (`CLIENT2_POOL`):** Leases `192.168.11.10` - `192.168.11.50`, gateway `192.168.11.1`, DNS `10.0.0.2`.
- **Router R2 Authoritative DNS:** Configured with `ip dns server` and static host mapping `server.[yourlastname].edu` -> `192.168.20.100`.

### 5.3 Deployment Strategy & Wireshark Trace Capture
- **CML Deployment Strategy:** Deploy `server.py` onto Subnet C node (`192.168.20.100`) behind Router R2, and `client.py` onto Subnet A and Subnet B nodes behind Router R1.
- **Cisco Infrastructure Configuration:** Router R1 DHCP pools (`CLIENT1_POOL`, `CLIENT2_POOL`) and Router R2 authoritative DNS (`ip host server.[lastname].edu 192.168.20.100`).
- **Wireshark Trace Capture Plan:** Capture DHCP DORA exchange (`dhcp_negotiation.pcap`) and DNS query/response resolution (`dns_lookup.pcap`).
