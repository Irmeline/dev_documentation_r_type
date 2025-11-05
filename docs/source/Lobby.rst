=====
Lobby
=====

The Lobby is the central hub of the R-Type server. It acts as the primary entry point for all clients, managing game sessions rather than the game simulation itself. Its main responsibilities are to handle new connections, manage a list of active game rooms, and redirect players to the appropriate game server instance.

This system runs on a fixed, public port (e.g., 4242), while each individual game runs on a separate, dynamically assigned port.

High-Level Architecture
-----------------------

1.  A **Client** connects to the **Lobby** on the main port.
2.  The client sends a request to either create or join a game room.
3.  The **Lobby** processes the request:
    - If a suitable public room exists, it adds the player.
    - If not, it creates a new **GameServer** instance.
4.  The `GameServer` instance starts listening on a **new, private port**.
5.  The **Lobby** sends a `JoinSuccessPacket` back to the client, containing the **new port** for the game.
6.  The **Client** disconnects from the lobby and reconnects to the dedicated game port to start playing.

This architecture provides excellent isolation between game sessions and prevents the lobby from becoming a bottleneck.

Key Classes
-----------

``Lobby``
  The main class that orchestrates the lobby. It listens for incoming UDP packets on the main server port and manages all `GameServer` instances. Its core logic resides in a packet-processing loop.

  .. code-block:: cpp
     :caption: Lobby.hpp (extrait)

     class Lobby {
     public:
         Lobby(asio::io_context& io_context, unsigned short lobby_port);
     
     private:
         void handleReceive(const asio::error_code& error, std::size_t bytes);
         void processLobbyPacket(const udp::endpoint& sender, const std::vector<char>& data);
         
         void handleCreateRoom(const udp::endpoint& host, const CreateRoomPacket& packet);
         void handleQuickJoin(const udp::endpoint& newPlayer);

         asio::io_context& _io_context;
         udp::socket _socket;
         std::map<int, RoomInfo> _rooms;
     };

``GameServer``
  An object representing a complete, independent R-Type game session. Each ``GameServer`` instance runs its own Engine loop in a separate thread and communicates on its own dedicated port.

  .. code-block:: cpp
     :caption: GameServer.hpp (extrait)

     class GameServer {
     public:
         GameServer(asio::io_context& io_context, unsigned short port, const std::string& levelPath);
         
         void start(); // Starts the game loop in a new thread
         void stop();
         unsigned short getPort() const;

     private:
         void gameLoop();

         Engine _engine;
         std::thread _gameThread;
         std::atomic<bool> _isGameRunning;
     };


Client Actions
--------------

The client communicates with the lobby using specific opcodes defined in the protocol.

.. list-table::
   :widths: 20 60
   :header-rows: 1

   * - OpCode (Client -> Server)
     - Description
   * - ``CREATE_ROOM``
     - Requests the creation of a new, private game room. The player is automatically placed inside as the host.
   * - ``CREATE_INFINITE``
     - Requests the creation of a private, single-player game room that starts immediately.

Server Responses
----------------

The lobby responds to client requests with specific packets.

.. list-table::
   :widths: 20 60
   :header-rows: 1

   * - OpCode (Server -> Client)
     - Description
   * - ``JOIN_SUCCESS``
     - Confirms that the player has successfully created or joined a room. **Crucially, this packet contains the new, dedicated game port** the client must connect to.
   * - ``JOIN_FAILURE``
     - Informs the client that they could not join a room (e.g., room is full).

.. code-block:: cpp
   :caption: Protocol.hpp (extrait)

   struct JoinSuccessPacket {
       uint8_t opcode = OpCode::JOIN_SUCCESS;
       unsigned short port;
   };


Example Workflow: Join
----------------------------

1.  **Client** sends a ``JOIN`` packet to the server on port **4242**.
2.  **Lobby** receives the packet and calls ``handleJoin``. It finds no available rooms.
3.  The ``Lobby`` creates and starts a new ``GameServer`` instance for **Room 0**, instructing it to listen on a new port, e.g., **4243**.

    .. code-block:: cpp
       :caption: Lobby::handleQuickJoin (extrait)

       if (targetRoomId == -1) {
           // No room found, create a new one. This implicitly calls handleCreateRoom.
           handleCreateRoom(newPlayer, false); // `false` for isPrivate
           return;
       }

4.  **Lobby** sends a ``JOIN_SUCCESS`` packet back to the client, containing ``{ port: 4243 }``.
5.  **Client** receives the packet, disconnects from the lobby, and reconfigures its `ClientNetworkSystem` to communicate on port **4243**.
6.  **Subsequent players** sending ``JOIN`` will be added to Room 0 until it has 4 players.
7.  When the **4th player** joins, the `Lobby` calls the final trigger:

    .. code-block:: cpp
       :caption: Lobby::handleQuickJoin (extrait)

       if (roomToJoin.players.size() == 4) {
           std::cout << "[Lobby] La Room " << targetRoomId << " est pleine ! Démarrage de la partie." << std::endl;
           
           // Gives the signal to the GameServer instance to start its Engine loop.
           roomToJoin.gameInstance->start();
       }