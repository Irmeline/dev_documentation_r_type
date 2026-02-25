=====
Lobby
=====

The Lobby is the central hub of the R-Type server. It acts as the primary entry point for all clients, managing game sessions rather than the game simulation itself. It handles new connections, manages a list of active game rooms, and redirects players to the appropriate ``GameServer`` instance.

The lobby runs on a fixed, configurable port (default: 3000), while each game room runs on a separate, dynamically assigned port.

High-Level Architecture
-----------------------

1. A **Client** sends a ``HELLO`` packet to the **Lobby** on the main port.
2. The Lobby responds with a ``ROOM_LIST`` packet containing all active multiplayer rooms.
3. The client sends a request to create or join a game room.
4. The **Lobby** processes the request:

   - ``CREATE_ROOM``: Creates a new multiplayer room. The player is the owner and waits for up to 3 more players.
   - ``JOIN_ROOM``: Joins an existing room by name. When the room reaches 4 players, the game starts.
   - ``CREATE_INFINITE``: Creates a single-player infinite mode room that starts immediately.

5. The ``GameServer`` instance starts listening on a **new, dedicated port**.
6. The **Lobby** sends a ``JOIN_SUCCESS`` or ``GAME_STARTING`` packet containing the game port.
7. The **Client** connects to the dedicated game port.

This architecture provides isolation between game sessions and prevents the lobby from becoming a bottleneck.

Room Types
----------

.. list-table::
   :widths: 20 40 15
   :header-rows: 1

   * - Type
     - Description
     - Max Players
   * - ``Room_Multiplayer``
     - Standard room. Owner creates it, others join. Game starts when full (4 players).
     - 4
   * - ``Solo_Room``
     - Infinite mode. Created and started immediately for a single player.
     - 1

Key Classes
-----------

``Lobby``
  The main class that orchestrates the lobby. It listens for incoming UDP packets and manages all ``GameServer`` instances.

``GameServer``
  An object representing a complete, independent R-Type game session. Each instance runs its own ``Engine`` loop in a separate thread and communicates on its own dedicated port.

``RoomInfo``
  A struct holding room state: name, type, player list, banned players, owner endpoint, and a ``unique_ptr<GameServer>`` for the game instance.

Client-to-Lobby Opcodes
------------------------

.. list-table::
   :widths: 20 60
   :header-rows: 1

   * - OpCode
     - Description
   * - ``HELLO``
     - Initial connection. Lobby registers the client and sends back the room list.
   * - ``CREATE_ROOM``
     - Creates a new multiplayer room. Contains ``roomName`` and ``playerPseudo``.
   * - ``JOIN_ROOM``
     - Joins an existing room by name. Contains ``roomName`` and ``playerPseudo``.
   * - ``CREATE_INFINITE``
     - Creates a solo infinite mode room. Contains ``playerPseudo``. Starts immediately.
   * - ``LEAVE_ROOM``
     - Leaves the current room. Contains ``roomName`` and ``playerPseudo``.
   * - ``ACK``
     - Acknowledges receipt of a reliable packet (by sequence number).
   * - ``ID``
     - Sent after a kick/ban to inform the lobby of the player's entity ID for cleanup.

Lobby-to-Client Opcodes
------------------------

.. list-table::
   :widths: 20 60
   :header-rows: 1

   * - OpCode
     - Description
   * - ``ROOM_LIST``
     - Compressed list of all active multiplayer rooms (name, player count, player names). Sent after HELLO, room changes, or joins.
   * - ``ROOM_CREATED``
     - Confirms room creation. Contains ``roomId``.
   * - ``PLAYER_JOINED``
     - Confirms the player has joined a room that is not yet full.
   * - ``GAME_STARTING``
     - Sent to all players in a room when it becomes full. Contains ``gamePort``.
   * - ``JOIN_SUCCESS``
     - Sent for infinite mode. Contains the dedicated ``gamePort``.
   * - ``JOIN_FAILURE``
     - Player cannot join (e.g., banned from the room).
   * - ``ROOM_FULL``
     - The requested room already has 4 players.
   * - ``KICKPLAYER``
     - Broadcast to all players in a room when a player is kicked.
   * - ``BANPLAYER``
     - Broadcast to all players in a room when a player is banned.

Reliable Delivery
-----------------

All lobby packets include a ``sequence`` number. The lobby stores pending reliable packets and retransmits them every second until an ``ACK`` is received from the client. This ensures that critical packets (``JOIN_SUCCESS``, ``GAME_STARTING``, ``ROOM_LIST``) are not lost.

Room List Compression
---------------------

The ``ROOM_LIST`` packet can be large (up to 16 rooms with 4 player names each). It is compressed with zlib before transmission. The final packet is: ``[ROOM_LIST opcode byte] + [zlib compressed payload]``.

Admin Console
-------------

The server provides an interactive admin console running in a separate thread. Available commands:

.. list-table::
   :widths: 25 50
   :header-rows: 1

   * - Command
     - Description
   * - ``rooms``
     - Print all active rooms with their ID, name, type, player count, and port.
   * - ``players``
     - Print all players in all rooms with their pseudo, endpoint, and owner status.
   * - ``kick <pseudo> <room>``
     - Remove a player from a room. A ``KICKPLAYER`` packet is broadcast to all players in the room.
   * - ``ban <pseudo> <room>``
     - Remove a player and add them to the room's ban list. A ``BANPLAYER`` packet is broadcast. The player cannot rejoin this room.
   * - ``stop``
     - Stop the server and all game instances.
   * - ``help``
     - Display available commands.

Example Workflow: Infinite Mode
-------------------------------

1. **Client** sends ``HELLO`` to lobby port 3000.
2. **Lobby** responds with ``ROOM_LIST`` (may be empty).
3. **Client** sends ``CREATE_INFINITE`` with pseudo.
4. **Lobby** creates a ``GameServer`` on port 3001, loads ``level_manager_infinite.lua``, and starts it immediately.
5. **Lobby** sends ``JOIN_SUCCESS { port: 3001 }`` to the client.
6. **Client** connects to port 3001 and begins gameplay.

Example Workflow: Multiplayer Room
----------------------------------

1. **Player A** sends ``CREATE_ROOM { roomName: "MyRoom", pseudo: "Alice" }``.
2. **Lobby** creates the room and sends ``ROOM_CREATED`` to Player A. Broadcasts updated ``ROOM_LIST`` to all lobby clients.
3. **Players B, C** send ``JOIN_ROOM { roomName: "MyRoom", pseudo: "Bob/Charlie" }``. Lobby sends ``PLAYER_JOINED`` to each. Updates room list.
4. **Player D** sends ``JOIN_ROOM``. Room is now full (4/4).
5. **Lobby** sends ``GAME_STARTING { gamePort: 3001 }`` to all 4 players. Calls ``GameServer::start()``.
6. All 4 clients connect to port 3001 and begin gameplay.