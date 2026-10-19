# Concurrency Design: Dots and Boxes Game Server

## 1. Architecture Decision

The server uses **non-blocking I/O multiplexing (`selectors.DefaultSelector()`)** rather than multi-threading.

- **Transport:** TCP
- **Concurrency model:** single-threaded event loop, non-blocking sockets (`setblocking(False)`)
- **Selector:** `selectors.DefaultSelector()`; maps file descriptors to `(events, data/callback)` for event serialization.
- **Synchronization primitives:** none. No `threading.Lock` is used because all shared-state access happens on one thread.

## 2. Shared Resources

The following shared resources exist in server memory and are accessed only within the event loop:

| Resource | Description |
|---|---|
| **Game board matrix** | `board.horizontal` (4×4), `board.vertical` (3×5), `board.boxes` (3×4) tracking drawn lines and box ownership |
| **Active turn state** | `turn` counter (increments on valid move), `current_player_id` (1 or 2) |
| **Connected client registry** | Map of fd → `{socket, player_id, player_name, buffer, state}` for both connections |
| **Round scores** | `scores.player_1`, `scores.player_2` (box ownership counts) |
| **Per-connection byte buffers** | Accumulator for partial TCP reads, split on `\n` delimiter per connection |
| **Selector registrations** | fd → `(fileobj, EVENT_READ, callback)` entries for the listening socket and each client socket |

All resources are mutated and read exclusively on the event loop thread.

## 3. Race-Condition Prevention

### 3.1 Single-Threaded Execution State Safety

The server runs one `while True: select()` loop on a single thread. Every shared resource is read and written only by that thread. Because no other thread can preempt between a state read and its dependent write, no read-modify-write race is possible.

Example atomic sequence within one event-loop iteration:
1. `recv()` on fd → append to that fd's byte buffer
2. Split buffer on `\n`, extract a complete JSON frame
3. Parse and validate the `PLAYER_TURN`
4. Check `current_player_id == msg.player_id`
5. Mutate board matrix
6. Update `turn` and `current_player_id`
7. Update scores
8. Broadcast `UPDATE_BOARD` to both clients
9. Return to `select()`

Steps 3–8 occur without yielding to another thread, so the board, turn, and scores are always mutually consistent when observed by any client.

### 3.2 Event-Driven Isolation

`select()` blocks until at least one registered socket is ready. When it returns, exactly one ready event (or a small batch) is dispatched to its callback. Callbacks run to completion before the next `select()` call. This means:

- **Accept isolation:** a new connection is fully accepted, registered, and its `CONNECT` awaited before another event is processed.
- **Read isolation:** one client's complete message (buffer split, JSON parse, validate, apply, broadcast) is handled before another client's socket is serviced.
- **Write isolation:** `UPDATE_BOARD` / `GAME_OVER` broadcasts are sent to both client sockets within the same event-loop iteration that applied the move, so all clients observe the same game-state sequence.

Because all I/O is non-blocking, no callback blocks the loop waiting on socket I/O; `recv()` and `send()` only fire when `select()` reports readiness.

## 4. Connection Lifecycle

### Phase A: Listening Socket Setup

1. Create `socket(AF_INET, SOCK_STREAM)`.
2. `setblocking(False)` on the listening socket.
3. `bind((HOST, PORT))`.
4. `listen()`.
5. `selector.register(listen_fd, EVENT_READ, accept_cb)`.

The listening socket remains registered for the lifetime of the server process.

### Phase B: Accept and Client Registration

When `select()` reports `EVENT_READ` on the listening socket:

1. `accept()` returns a new non-blocking client socket `conn`.
2. `selector.register(conn_fd, EVENT_READ, read_cb)`.
3. Insert into the client registry: `{socket: conn, player_id: None, player_name: None, buffer: b"", state: "AWAITING_CONNECT"}`.
4. Wait for the client's `CONNECT` message (see Phase C).
5. On valid `CONNECT`:
   - Assign `player_id` (1 for first, 2 for second).
   - Set registry `player_id` and `player_name`.
   - Send `LOBBY_WAIT {player_id: 1}` to the first player.
6. When both players have connected:
   - Initialize empty board matrix.
   - Set `turn = 1`, `current_player_id = 1`, scores to 0.
   - Broadcast `BEGIN {players, board, turn, current_player_id}` to both clients.

If a second client connects while a game is in progress, the server follows the disconnect/forfeit rules (Phase D) or rejects per game-membership authority (the server is authoritative for game membership, per `protocol_blueprint.md`).

### Phase C: Active Game Loop (Event-Driven Servicing)

This is the steady state. On each `select()` wake-up:

#### 1. Read readiness on a client socket

1. `conn.recv(4096)`.
2. If `data == b""` → orderly shutdown; branch to Phase D teardown.
3. Append `data` to that fd's per-connection byte buffer.
4. Enforce the 64 KiB maximum frame size: if the buffer exceeds 65,536 bytes without finding `\n`, close the connection.
5. Search the buffer for the first `\n` (0x0A).
6. Extract all bytes before the delimiter as one complete frame.
7. Decode as UTF-8; on invalid UTF-8, send `ERROR` and/or close.
8. Parse as JSON; on invalid JSON or non-object top-level value, send `ERROR`.
9. Validate the JSON schema (`protocol_version`, `msg_type`, `msg_id`, `timestamp`, `payload`).
10. Process the message:
    - **`PLAYER_TURN`**: validate `player_id == current_player_id`; validate coordinates (adjacent, in bounds, undrawn line); apply line to board matrix; check for box completion; update scores; increment `turn`; toggle `current_player_id`; broadcast `UPDATE_BOARD` to both clients. If all boxes are claimed, broadcast `GAME_OVER` to both.
    - **`QUERY_BOARD`**: respond with `UPDATE_BOARD` reflecting current state.
    - Other types: handle per their schema or send `ERROR`.
11. Continue processing any additional complete frames remaining in the buffer.
12. Return to `select()`.

#### 2. Write readiness (broadcast)

`UPDATE_BOARD` and `GAME_OVER` broadcasts are sent immediately after state mutation within the same event-loop iteration. `send()` is used on non-blocking sockets; if `send()` would block (`BlockingIOError`), the data is retained and retried on the next `EVENT_WRITE` notification (registration can be temporarily modified to include `EVENT_WRITE` if needed).

### Phase D: Client Socket Teardown

Teardown is triggered by:
- `recv()` returning `b""` (orderly shutdown), or
- A socket exception.

#### Teardown steps (executed on the event loop thread):

1. `handle_disconnect(conn)`:
   - Discard any incomplete frame remaining in that fd's byte buffer (it is not a valid JSON frame).
   - `selector.unregister(conn_fd)`.
   - `conn.close()`.
   - Remove the entry from the client registry.
2. Determine game impact:
   - If the disconnect occurs **during a game** (both players connected, `BEGIN` already sent): the remaining player wins by forfeit. Broadcast `GAME_OVER {reason: FORFEIT, winner_id: <remaining player id>}` to the remaining client.
   - If the disconnect occurs **in the lobby** (only one player connected, game not started): the remaining player stays in lobby waiting (`LOBBY_WAIT` semantics); no `GAME_OVER` is sent until a game begins; forfeit will be broadcast to Player 2 once game has begun.
3. Return to `select()` to service remaining clients.

## 5. Turn Enforcement

The server is authoritative for the active player. Before applying any `PLAYER_TURN`:

1. Read `current_player_id` from shared turn state.
2. Reject the move with `ERROR` if `msg.player_id != current_player_id`.
3. Reject the move with `ERROR` if coordinates are malformed, non-adjacent, out of bounds, or reference an already-drawn line.
4. Only after all validation passes does the server mutate the board matrix, update scores, increment `turn`, and toggle `current_player_id`.

Because steps 1–4 occur atomically in one event-loop iteration, a player cannot submit a move out of turn even if both clients send `PLAYER_TURN` simultaneously; only the first one processed by `select()` will see `current_player_id` matching its own ID.

## 6. Summary

| Concern | Mechanism |
|---|---|
| Concurrency | `selectors.DefaultSelector()` event loop, non-blocking sockets |
| Shared resources | Board matrix, turn state, client registry, scores, byte buffers, selector map; all single-threaded |
| Race prevention | Single-threaded execution state safety and event-driven isolation (no locks) |
| Accept path | `accept()` → non-blocking → `register()` → await `CONNECT` → assign ID → `BEGIN` broadcast |
| Game loop | `select()` → `recv()` → buffer → split on `\n` → validate → mutate → broadcast → `select()` |
| Teardown | `recv() == b""` → discard incomplete frame → `unregister()` → `close()` → registry removal → `GAME_OVER {FORFEIT}` to remaining player if mid-game |
| Turn enforcement | `current_player_id` checked before any board mutation, atomically per event |
| Frame safety | Per-conn buffer, `\n` delimiting, 64 KiB limit, UTF-8/JSON validation |
