============
Architecture
============

This section describes the high-level architecture of the R-Type project, the folder structure, and how the different modules interact.

Project Structure
-----------------

.. code-block:: text

   R-Type/
   ├── engine/
   │   ├── include/
   │   │   ├── core/
   │   │   │   ├── registry.hpp              # ECS Registry (sparse arrays, entities)
   │   │   │   ├── sparse_array.hpp           # Contiguous component storage
   │   │   │   ├── entity.hpp                 # Entity type alias
   │   │   │   ├── PluginLoader.hpp           # Dynamic backend loading
   │   │   │   ├── RessourceManager.hpp       # Asset loader (textures, fonts, sounds)
   │   │   │   ├── client_entity_factory.hpp  # JSON-driven entity creation (client)
   │   │   │   ├── server_entity_factory.hpp  # JSON-driven entity creation (server)
   │   │   │   ├── Components/
   │   │   │   │   ├── client_components.hpp  # SpriteComponent, TextComponent, etc.
   │   │   │   │   └── server_components.hpp  # Position, Velocity, Health, Player, etc.
   │   │   │   └── Systems/
   │   │   │       ├── Engine.hpp             # System pipeline + game loop
   │   │   │       ├── ISystem.hpp            # Abstract system interface
   │   │   │       ├── InputSystem.hpp        # Keyboard input → events
   │   │   │       ├── RenderSystem.hpp       # Draws sprites + texts
   │   │   │       ├── UISystem.hpp           # HUD, overlays, game over, victory
   │   │   │       ├── Animation.hpp          # Sprite sheet animation
   │   │   │       ├── scrollingSystem.hpp    # Parallax backgrounds
   │   │   │       ├── audio.hpp              # Sound playback
   │   │   │       ├── Movements.hpp          # Position += Velocity * dt
   │   │   │       ├── Collision.hpp          # AABB collision + damage
   │   │   │       ├── Weapon.hpp             # Bullet creation from ShootEvents
   │   │   │       ├── ForcePod.hpp           # Force Pod attachment + shooting
   │   │   │       ├── Ennemy_ai.hpp          # Enemy behaviors (straight, sin, etc.)
   │   │   │       ├── Game_rulesSystems.hpp  # Lua scripting bridge
   │   │   │       ├── StayW.hpp              # Keep entities within screen bounds
   │   │   │       └── GameLogicSystem.hpp    # Local player input event
   │   │   ├── graphics/
   │   │   │   ├── IGraphicFactory.hpp        # Abstract factory
   │   │   │   ├── IRenderWindow.hpp          # Window + event polling
   │   │   │   ├── IRenderer.hpp              # Draw calls (sprites, text, shapes)
   │   │   │   ├── Isprite.hpp                # Drawable textured object
   │   │   │   ├── ITexture.hpp               # Image data
   │   │   │   ├── IFont.hpp                  # Font data
   │   │   │   ├── IText.hpp                  # Drawable text
   │   │   │   ├── Isound.hpp                 # Sound playback
   │   │   │   ├── IsoundBuffer.hpp           # Sound data
   │   │   │   └── IEvents.hpp                # Event types, key codes
   │   │   └── network/
   │   │       ├── protocol.hpp               # Opcodes, packet structs
   │   │       ├── serialisation.hpp          # Binary read/write helpers
   │   │       ├── Datacompression.hpp        # zlib compression
   │   │       ├── Event.hpp                  # EventBus + game events
   │   │       ├── GameConfig.hpp             # EntityType enum + name mapping
   │   │       ├── Server_NetworkSyncSystems.hpp
   │   │       └── Client_NetworkSyncSystems.hpp
   │   └── src/                               # .cpp implementations
   │
   ├── Server/
   │   ├── include/
   │   │   ├── Lobby.hpp                      # Room management
   │   │   ├── GameServer.hpp                 # One game instance per room
   │   │   └── GameManage.hpp                 # Input packet processor
   │   ├── src/
   │   │   ├── main.cpp                       # Lobby + admin console
   │   │   ├── network/
   │   │   │   ├── Lobby.cpp
   │   │   │   ├── LobbyDashboard.cpp         # kick, ban, rooms, players
   │   │   │   ├── GameServer.cpp
   │   │   │   ├── GameManage.cpp
   │   │   │   └── LoggerGlobal.cpp
   │   │   └── ...
   │   ├── config_files/                      # Server-side entity JSON templates
   │   └── levels/                            # Lua level scripts
   │       ├── level_manager.lua              # 4-level campaign progression
   │       ├── level_manager_infinite.lua     # Infinite mode manager
   │       ├── level0.lua                     # Infinite mode (endless waves)
   │       ├── level1.lua                     # Level 1: Asteroid Field
   │       ├── level2.lua                     # Level 2: Enemy Territory
   │       ├── level3.lua                     # Level 3: Alien Hive
   │       └── level4.lua                     # Level 4: Final Assault
   │
   ├── Client/
   │   ├── include/
   │   │   ├── menuSystem.hpp                 # Lobby UI + room browser
   │   │   └── renderMenu.hpp                 # Menu rendering
   │   ├── src/
   │   │   └── main.cpp                       # Session loop (menu ↔ game ↔ cleanup)
   │   ├── config_files/                      # Client-side entity JSON templates
   │   ├── assets/                            # Sprites, fonts, sounds
   │   └── resources.json                     # Asset catalogue
   │
   └── Backends/
       └── SFMLBackend/                       # SFML implementation of all interfaces
           ├── include/
           └── src/


ECS Architecture
----------------

The engine uses an **Entity Component System** architecture where:

- **Entities** are simple numeric IDs (``size_t``).
- **Components** are plain data structs stored in ``sparse_array<T>`` containers inside the ``Registry``.
- **Systems** are classes implementing ``ISystem::update(Registry&, float)`` that operate on components each frame.

.. code-block:: text

   Registry
   ├── sparse_array<Position>      [0: {100,200}, 1: {500,300}, 2: nullopt, ...]
   ├── sparse_array<Velocity>      [0: {-200,0},  1: {0,0},     2: nullopt, ...]
   ├── sparse_array<Health>        [0: {10},       1: {100},     2: nullopt, ...]
   ├── sparse_array<Player>        [0: nullopt,    1: {score:0}, 2: nullopt, ...]
   └── sparse_array<AI_enemy>      [0: {STRAIGHT}, 1: nullopt,   2: nullopt, ...]

The ``Engine`` class holds a ``Registry`` and a list of ``ISystem`` pointers. Each frame, it calls ``update()`` on every system in order.


Server System Execution Order
-----------------------------

The order of systems in the server pipeline is critical:

.. code-block:: text

   1. GameRulesSystem     — Lua script: spawns enemies, triggers events
   2. AISystem            — Enemy behaviors, publishes ShootEvents
   3. ForcePodSystem      — ForcePod shooting (synchronized with player)
   4. WeaponSystem        — Reads ShootEvents, creates bullet entities
   5. MovementSystem      — Position += Velocity * deltaTime
   6. CollisionSystem     — AABB detection, damage, powerups, destroy
   7. StayWSystem         — Clamp entities within screen bounds
   8. NetworkSystem       — Broadcast snapshot, reset velocities, clear events


Client System Execution Order
-----------------------------

.. code-block:: text

   1. InputSystem         — Reads keyboard, publishes LocalPlayerInputEvent
   2. UISystem            — Reads UI events, updates HUD and overlays
   3. NetworkSystem       — Sends inputs, receives snapshots, clears events
   4. ScrollingSystem     — Scrolls background entities
   5. AnimationSystem     — Advances sprite sheet frames
   6. RenderSystem        — Draws all sprites + UI texts (LAST)


Data Flow
---------

.. code-block:: text

   Client                          Server
   ══════                          ══════
   InputSystem                     Lobby (port 3000)
       │ KeyPressed                    │ CREATE_ROOM / JOIN / INFINITE
       ▼                               ▼
   NetworkSystem ──── UDP ────► GameServer (port 3001+)
       │ CLIENT_INPUT                  │
       │                               ▼
       │                          GameManage.handleIncomingPacket()
       │                               │ sets Velocity, wantsToShoot
       │                               ▼
       │                          ECS Pipeline (8 systems)
       │                               │
       │                               ▼
       │                          NetworkSystem.broadcastWorldState()
       │                               │ pre-filter, compress, send
       ◄────────── UDP ────────────────┘
       │ GAME_SNAPSHOT (compressed)
       ▼
   decompress → parse header → apply destructions/creations/modifications
       │
       ▼
   UISystem reads PlayerStateUIEvent → updates HUD
       │
       ▼
   RenderSystem draws everything