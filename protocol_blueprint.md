# Dots aNd Boxes Protocol (DNBP) Specification v1.0
---
## Fields

All DNBP packets must include the following fields:

| Field | JSON type |  Rules |
|---|---|
| `protocol_version` | string |  Exactly `"1.0"` |
| `msg_type` | string |  One of the defined message types |
| `msg_id` | string | Unique message identifier |
| `timestamp` | integer | Unix epoch timestamp in seconds |
| `payload` | object |  |

---
## Message Types:
### 1. `CONNECT`

**Direction:** Client → Server  
**Purpose:** Request to join the game.

| Field | JSON type | Description |
|---|---|---|
| `player_name` | string | Display name requested by the client |

*Example:*
```json
{
  "protocol_version": "1.0",
  "msg_type": "CONNECT",
  "msg_id": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1760000000,
  "payload": {
    "player_name": "Alice"
  }
}
```
### 2. `LOBBY_WAIT`

**Direction:** Server → Client  
**Purpose:** Notify the first player that the server is waiting for Player 2.

| Field | JSON type | Description |
|---|---|---|
| `player_id` | integer | Assigned player ID; normally `1` |
| `player_name` | string | Name accepted for the connected player |
| `notification` | string | A welcome string addressed to Player 1 |

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "LOBBY_WAIT",
  "msg_id": "650e8400-e29b-41d4-a716-446655440001",
  "timestamp": 1760000001,
  "payload": {
    "player_id": 1,
    "player_name": "Alice",
    "waiting_for_player_id": 2
  }
}
```

### 3. `BEGIN`

**Direction:** Server → Clients  
**Purpose:** Start the game and broadcast the initial game state.

| Field | JSON type | Description |
|---|---|---|
| `players` | array | Players participating in the game |
| `players[].player_id` | integer | Server-assigned ID, either `1` or `2` |
| `players[].player_name` | string | Player display name |
| `board` | object | Empty initialized board |
| `turn` | integer | Current turn number; starts at `1` |
| `current_player_id` | integer | Player whose turn it is |

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "BEGIN",
  "msg_id": "750e8400-e29b-41d4-a716-446655440002",
  "timestamp": 1760000002,
  "payload": {
    "players": [
      {
        "player_id": 1,
        "player_name": "Alice"
      },
      {
        "player_id": 2,
        "player_name": "Bob"
      }
    ],
    "board": {
      "horizontal": [
        [0, 0, 0, 0],
        [0, 0, 0, 0],
        [0, 0, 0, 0],
        [0, 0, 0, 0]
      ],
      "vertical": [
        [0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0]
      ],
      "boxes": [
        [null, null, null, null],
        [null, null, null, null],
        [null, null, null, null]
      ]
    },
    "turn": 1,
    "current_player_id": 1
  }
}
```

### 4. `PLAYER_TURN`

**Direction:** Client → Server  
**Purpose:** Submit one line drawn by a player.

| Field | JSON type | Description |
|---|---|---|
| `player_id` | integer | Player making the move; must be `1` or `2` |
| `start` | array | First coordinate pair `[row, column]` |
| `end` | array | Second coordinate pair `[row, column]` |

The coordinates must:
- Contain exactly two integers.
- Be within rows `0–3` and columns `0–4`.
- Be horizontally or vertically adjacent.
- Refer to a line that has not already been drawn.

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "PLAYER_TURN",
  "msg_id": "850e8400-e29b-41d4-a716-446655440003",
  "timestamp": 1760000010,
  "payload": {
    "player_id": 1,
    "start": [0, 0],
    "end": [0, 1]
  }
}
```

### 5. `UPDATE_BOARD`

**Direction:** Server → Clients  
**Purpose:** Broadcast the updated board after a valid move.

| Field | JSON type | Description |
|---|---|---|
| `board` | object | Updated board state |
| `scores` | object | Current scores |
| `scores.player_1` | integer | Player 1 score |
| `scores.player_2` | integer | Player 2 score |
| `turn` | integer | Current turn number |
| `current_player_id` | integer | Player whose turn is next |

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "UPDATE_BOARD",
  "msg_id": "950e8400-e29b-41d4-a716-446655440004",
  "timestamp": 1760000011,
  "payload": {
    "board": {
      "horizontal": [
        [1, 0, 0, 0],
        [0, 0, 0, 0],
        [0, 0, 0, 0],
        [0, 0, 0, 0]
      ],
      "vertical": [
        [0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0],
        [0, 0, 0, 0, 0]
      ],
      "boxes": [
        [null, null, null, null],
        [null, null, null, null],
        [null, null, null, null]
      ]
    },
    "scores": {
      "player_1": 0,
      "player_2": 0
    },
    "turn": 2,
    "current_player_id": 2
  }
}
```

### 6. `QUERY_BOARD`

**Direction:** Client → Server  
**Purpose:** Request the current board and turn state (response is `UPDATE_BOARD`).

| Field | JSON type | Description |
|---|---|---|
| `player_id` | integer | Requesting player; must be `1` or `2` |

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "QUERY_BOARD",
  "msg_id": "a50e8400-e29b-41d4-a716-446655440005",
  "timestamp": 1760000015,
  "payload": {
    "player_id": 2
  }
}
```

### 7. `GAME_OVER`

**Direction:** Server → Clients  
**Purpose:** Notify clients that the game has ended.

| Field | JSON type | Description |
|---|---|---|
| `reason` | string | One of `"VICTORY"`, `"DRAW"`, or `"FORFEIT"` |
| `winner_id` | integer or null | Winning player, or `null` for a draw |
| `scores` | object | Final scores |
| `scores.player_1` | integer | Final Player 1 score |
| `scores.player_2` | integer | Final Player 2 score |
| `board` | object | Final board state |
| `turn` | integer | Final turn number |

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "GAME_OVER",
  "msg_id": "b50e8400-e29b-41d4-a716-446655440006",
  "timestamp": 1760000100,
  "payload": {
    "reason": "VICTORY",
    "winner_id": 1,
    "scores": {
      "player_1": 7,
      "player_2": 5
    },
    "board": {
      "horizontal": [
        [1, 1, 1, 1],
        [1, 1, 1, 1],
        [1, 1, 1, 1],
        [1, 1, 1, 1]
      ],
      "vertical": [
        [1, 1, 1, 1, 1],
        [1, 1, 1, 1, 1],
        [1, 1, 1, 1, 1]
      ],
      "boxes": [
        [1, 2, 1, 1],
        [2, 1, 2, 1],
        [1, 1, 2, 1]
      ]
    },
    "turn": 38
  }
}
```

### 8. `ERROR`

**Direction:** Server → Client  
**Purpose:** Report a malformed packet or invalid move.

| Field | JSON type | Description |
|---|---|---|
| `code` | string | Error code for identification |
| `message` | string | User friendly explanation |

*Example*
```json
{
  "protocol_version": "1.0",
  "msg_type": "ERROR",
  "msg_id": "d50e8400-e29b-41d4-a716-446655440008",
  "timestamp": 1760000020,
  "payload": {
    "code": "NON_ADJACENT_POINTS",
    "message": "The submitted coordinates must identify adjacent dots."
  }
}
```
---
---
## Framing Rules

Each message is encoded as a UTF-8 JSON object + LF byte. The delimiter is one ASCII line-feed byte: `0x0A`

*Example byte-level framing:*
```text
{"msg_type":"QUERY_BOARD",...}\n{"msg_type":"PLAYER_TURN",...}\n
```

**Serialization Rules**
1. Each frame must contain exactly one complete JSON object.
2. JSON must be encoded as UTF-8.
3. JSON must be serialized compactly on one physical line.
4. A frame must end with `\n`.
5. The sender must *not* transmit a standalone blank line.
6. JSON whitespace outside strings may be minimized but must not include a physical newline.
7. Newlines inside JSON strings must be escaped as `\\n`.
8. The receiver must parse each complete line independently.
9. TCP packet boundaries must not be treated as message boundaries.

**Maximum Frame Size:** 64 KiB = 65,536 bytes, including the terminating newline.
If the receiver accumulates more than 65,536 bytes without finding `\n`, it must close the connection or return `ERROR`.

**The receiver must:**
1. Maintain a per-connection byte buffer.
2. Append incoming TCP bytes to that buffer.
3. Search for the first delimiter: `0x0A`.
4. Extract all bytes before the delimiter.
5. Decode the extracted bytes as UTF-8.
6. Parse the result as JSON.
7. Validate that the result is a JSON object.
8. Validate the JSON schema.
9. Process the message.
10. Continue processing any additional complete frames in the buffer.

**A receiver must support:**
- Partial messages split across TCP packets.
- Multiple messages contained in one TCP packet.
- Multiple messages read in a single socket operation.

The server should send `ERROR` when possible for:
- Invalid UTF-8.
- Invalid JSON.
- Missing newline before the frame-size limit.
- A top-level JSON value that is not an object.
- An empty frame.
- Unsupported protocol version.

The server may immediately close the connection for malformed or oversized input if necessary.

**Ordering and delivery:**
- The server must process valid messages from a connection in receive order.
- `msg_id` must be unique per client session.
- The server should track processed `msg_id` values to reject duplicate submissions.
- A client should wait for the server’s response before submitting another `PLAYER_TURN`.
- The server should serialize broadcasts so all clients observe the same game-state sequence.

## Connection Termination and Socket Lifecycle Management

A connection follows this general lifecycle:

1. Accept the socket.
2. Wait for and validate `CONNECT`.
3. Assign the player ID.
4. Receive and process framed messages.
5. Remove the player and close the socket when the connection ends, or upon GAME_OVER.

Before processing each new turn, the server must verify that both players are still connected. If one player disconnects during a game, the remaining player wins by forfeit through `GAME_OVER`. If the game has not started, the remaining player will follow through the same flow.

The will handle a separate disconnect message. A graceful application shutdown should stop sending messages, close the socket, remove the player from the connection registry, and apply the appropriate lobby or forfeit rules.

A TCP `recv()` call returning `b""` means the peer performed an orderly shutdown:

```python
data = sock.recv(4096)

if data == b"":
    handle_disconnect(connection)
    return
```

This must be treated as connection termination, not as an empty JSON frame. Any incomplete frame remaining in the buffer should be discarded. A frame containing only `\n` is an empty protocol frame and should produce an `ERROR` when possible. Connection-related exceptions should use the same cleanup path.

## Additional Notes

The server must be authoritative for:
- Player Number assignment.
- Game membership.
- Current board state.
- Current turn number.
- Active player.
- Scores.
- Game completion.
- Forfeits and disconnect handling.

A client must treat `BEGIN`, `UPDATE_BOARD`, and `GAME_OVER` as authoritative state snapshots.
A `PLAYER_TURN` is not an acknowledgement that a move succeeded. The move is accepted only when the server broadcasts `UPDATE_BOARD`.
