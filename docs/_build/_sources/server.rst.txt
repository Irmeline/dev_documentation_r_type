.. RTYPE SERVER documentation master file, created by
   sphinx-quickstart on Mon Oct 13 01:56:51 2025.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

RTYPE SERVER
============

This part helps developers understand the RTYPE server project from a high-level perspective.
It includes architectural diagrams, system overviews, networking details, tutorials, contribution guidelines,
and a technical comparative study of the technologies used.

Architecture Overview
=====================

The RTYPE server is built using a layered architecture typical of networked video games:

- **GameServer**: Manages the main loop and updates.
- **gameManage**: Handles networking, player connections, and entity logic.
- **EntityUpdateBroadcaster**: Sends entity snapshots to clients.
- **Registry**: ECS system managing entities and components.


Layered Subsystem Diagram
-------------------------
::  

    +---------------------+
    |     GameServer      |  <-- Main loop, threading, update()
    +---------------------+
              |
              v
    +---------------------+
    |     gameManage      |  <-- Handles networking, player input, spawning
    +---------------------+
              |
              v
    +---------------------+
    |      Registry       |  <-- ECS: manages entities & components
    +---------------------+
       |         |         |
       v         v         v
    Movement  Collision  EntityUpdateBroadcaster
     System     System         |
                               v
                        +------------------+     +-------+
                        |   UDP Broadcast  | --> |Client |
                        +------------------+     +-------+

   

Main Systems
============

Server Construction and Initialization
======================================
The RTYPE server is initialized through the `GameServer` constructor,
which sets up the core systems and prepares the server to run.

GameServer Constructor
----------------------

.. code-block:: cpp

   GameServer::GameServer(short port)
       : game(io, port), running(true) {
       std::cout << "[SERVER] Initialized on port " << port << std::endl;
   }

What it does:

- Initializes the `asio::io_context` object (`io`) for asynchronous networking.
- Instantiates the `gameManage` object with the I/O context and port.
- Sets the `running` flag to `true` to control the main loop.
- Logs the server startup.


gameManage Constructor
----------------------

.. code-block:: cpp

   gameManage::gameManage(asio::io_context& context, short port)
       : module(context, port) {}

What it does:

- Initializes the `UdpModule`, which handles low-level UDP socket communication.
- Prepares the server to listen for incoming packets from clients.

Startup Sequence
----------------

1. `main()` parses the port number and creates a `GameServer` instance.
2. `GameServer::run()` is called:
   - Starts listening for packets via `game.listen(registre)`
   - Launches a thread to run `io.run()` (Asio event loop)
   - Enters the main update loop (60 FPS)
3. Each frame, `update()` is called:
   - Runs movement and collision systems
   - Sends entity updates to all clients

This construction ensures that the server is ready to handle
real-time multiplayer interactions efficiently and safely.


**GameServer**

- The `GameServer`` class is the central controller of the server.
  it has its own level at which everything runs. It:

- Initializes the server with a given port;
- Starts the network listener via "game.listen(register)";
- Launches a dedicated thread to execute the Asio I/O context (`io.run()`).
- Executes the game loop and update()
- Calls the movement and collision systems, then sends entity updates to the clients.

.. code-block:: cpp

   void GameServer::run() {
       game.listen(registre);
       networkThread = std::thread([this]() { io.run(); });
       while (running) {
           update();
           std::this_thread::sleep_for(std::chrono::milliseconds(16));
       }
       io.stop();
       if (networkThread.joinable())
           networkThread.join();
   }

**gameManage**

- Handles incoming packets and player inputs.
- Spawns entities and broadcasts updates.
- Manages player connections and disconnections.

**EntityUpdateBroadcaster**

- Collects entity data from the Registry.
- Sends snapshots to all connected clients.
- Flags dead entities and ensures consistency.

**Registry**

- Manages all game entities and their components.
- ECS architecture for modularity and performance.


Networking Protocol
===================

The server uses UDP via the Asio library. Clients send input packets, and the server responds with updates.

Packet Types:

+------------------+--------+-------------------------------+
| Name             | Opcode | Description                   |
+==================+========+===============================+
| WelcomePacket    | 0x01   | Sent to new players           |
| PlayerInput      | 0x02   | Movement or shooting          |
| EntityUpdate     | 0x03   | Broadcast of entity states    |
| EntitySpawn      | 0x04   | New entity creation           |
| EntityDestroy    | 0x05   | Entity removal                |
| GameOver         | 0x06   | End of game notification      |
+------------------+--------+-------------------------------+

Robustness part:
================

- Server is multithreaded and non-blocking.
- Clients can crash without affecting server stability.
- Dead entities are flagged and broadcasted.


To ensure robustness and clarity when dealing with runtime issues, 
the RTYPE server uses a custom error class called `Error`, which extends `std::exception`.

Class Definition
----------------

.. code-block:: cpp

   class Error : public std::exception {
       private:
           std::string message;
       public:
           Error(const std::string &mess) : message(mess) {}
           const char* what() const throw () {
               return message.c_str();
           }
           Error() = default;
           ~Error() = default;
   };

Explanation
-----------

- The `Error` class is a lightweight wrapper around `std::exception`.
- It stores a custom error message and overrides the `what()` method to return it.
- This allows the server to throw meaningful exceptions and catch them gracefully.

Usage Example
-------------

.. code-block:: cpp

   void gameManage::verificate_size(size_t actual_len, size_t len) {
       try {
           if (actual_len > len)
               throw Error("Invalid packet received!!");
       } catch (const Error &e) {
           std::cerr << "[ERROR] " << e.what() << std::endl;
           return;
       }
   }

Benefits
--------

- Centralized error reporting
- Clear and readable error messages
- Prevents crashes due to malformed packets or unexpected input
- Keeps the server stable and maintainable



Tutorials and How-To's
=======================

**Run the server:**

.. code-block:: bash

   ./rtype_server 3000

**Connect a client:**

- Send `WelcomePacket` (opcode 0x00)
- Receive entity ID and position

**Spawn an entity:**

- Server creates entity with Position, Velocity, Health
- Sends `EntitySpawn` packet to clients

**Handle player input:**

- Movement: `actionType = 0`, direction 0-4
- Shooting: `actionType = 1`, creates bullet entity


Contribution Guidelines
=======================

Coding conventions:

- Functions: `camelCase`
- Classes: `PascalCase`
- ECS components: `Position`, `Velocity`, `Health`, etc.

Project structure:

- `GameServer`: main loop
- `gameManage`: networking and entity logic
- `EntityUpdateBroadcaster`: snapshot builder
- `Packets.hpp`: packet definitions

Best practices:

- Verify packet size with `verificate_size()`
- Use ECS for modularity
- Avoid blocking operations



Technical and Comparative Study
===============================

Algorithms and Data Structures
------------------------------

- ECS (Entity Component System) for modularity and performance
- `unordered_map` for fast access to entities and components
- Snapshot system for efficient network updates

Storage
-------

The RTYPE server uses **in-memory storage** via the ECS `Registry`. 
This choice is ideal for real-time games but comes with trade-offs.

- **Persistence**: No long-term persistence is implemented in the prototype. 
                  All entities exist only during runtime.
- **Reliability**: Memory-based storage is fast but volatile.
                  If the server crashes, all state is lost.
- **Constraints**:
  - Limited by RAM usage
  - No disk I/O or database integration
  - Suitable for short game sessions and fast-paced updates

Future improvements could include:
- Saving snapshots to disk
- Using a lightweight database (e.g., SQLite) for player stats or match history
- Implementing rollback or recovery mechanisms

Security
--------

Security and data integrity are handled through multiple layers:

- **Packet validation**: 
   All incoming packets are verified using `verificate_size()` to prevent buffer overflows or malformed data.
- **Entity flags**: 
   Dead or invalid entities are marked and excluded from updates.
- **Error module**: 
   Custom `Error` class ensures exceptions are caught and logged without crashing the server.

Vulnerabilities considered:

- **UDP spoofing**: mitigated by tracking client keys and IPs
- **Malformed packets**: handled via strict size checks and opcode validation
- **Denial of service**: server is multithreaded and non-blocking to avoid overload