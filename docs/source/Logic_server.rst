===============================
Creating a Game with the Engine
===============================

This guide explains how to use the core Engine to create a new game, using R-Type as an example. It covers the data-driven workflow, from defining entities in JSON to scripting level behavior with Lua.

The fundamental principle is **decoupling data from code**. Most of the game's content is defined in external configuration files, not in C++.

Workflow Overview
-----------------

Creating a game involves four main steps:

1. **Define Entity Blueprints**: Create JSON files that describe the components of each entity type.
2. **Script Level Behavior**: Create Lua scripts that define the sequence of events in a level.
3. **Implement Game Systems**: Write C++ classes (``ISystem``) for core game logic (collisions, AI, etc.).
4. **Assemble and Run**: Write a ``GameServer`` class that creates an ``Engine`` instance, adds all systems, and launches the game loop.

---

1. Entity Configuration (JSON Blueprints)
=========================================

Every entity type is defined as a template in a ``.json`` file. The ``ServerEntityFactory`` reads these files at startup.

Location
--------
The path to the configuration folder is passed to the factory constructor. Example: ``../Server/config_files/``.

Structure
---------
Each JSON file contains a ``components`` object. Each key is a component name, and its value contains the initial data.

Example: Enemy
--------------

.. code-block:: json
   :caption: config_files/enemy1.json

   {
     "components": {
       "Health": { "hp": 10 },
       "Hitbox": { "width": 66, "height": 66 },
       "Velocity": { "dx": -200.0, "dy": 0 },
       "AI_enemy": {
         "behavior": "STRAIGHT_SHOOTER",
         "scoreValue": 100,
         "moveSpeed": 200,
         "shootPeriodTicks": 120
       }
     }
   }

Example: Player Bullet
----------------------

.. code-block:: json
   :caption: config_files/player_bullet.json

   {
     "components": {
       "Velocity": { "dx": 1000.0, "dy": 0 },
       "Hitbox": { "width": 32, "height": 8 },
       "Damage": { "value": 10 },
       "Bullet": {}
     }
   }

To create a new entity type, simply create a new JSON file with the desired components.

---

2. Level Scripting (Lua)
========================

Level flow is controlled by Lua scripts. The ``GameRulesSystem`` loads and executes these scripts via the sol2 binding library.

Location
--------
``Server/levels/``

How It Works
------------
The ``GameRulesSystem`` creates a Lua virtual machine and binds several C++ functions. The script must define a global ``update(deltaTime)`` function, called every game tick.

Bound Functions
---------------

.. list-table::
   :widths: 35 65
   :header-rows: 1

   * - Function
     - Description
   * - ``create_entity(template, x, y)``
     - Creates an entity from a JSON blueprint at position (x, y).
   * - ``get_random_position(min, max)``
     - Returns a random float between min and max.
   * - ``get_player_score(playerId)``
     - Returns the current score of a player entity.
   * - ``get_all_player_ids()``
     - Returns a table of all connected player entity IDs.
   * - ``notify_level_completed()``
     - Publishes a ``LevelCompletedEvent`` to the EventBus.
   * - ``load_level(scriptPath)``
     - Loads and executes a new Lua script (used by the level manager).
   * - ``spawn_powerup(type, x, y)``
     - Creates a powerup entity of the given type.
   * - ``spawn_boss(template, x, y, name)``
     - Creates a boss entity and registers it for tracking.
   * - ``is_boss_alive()``
     - Returns true if the current boss entity still exists.
   * - ``notify_warning(message)``
     - Publishes a ``WarningEvent`` for client-side display.
   * - ``load_victory_screen()``
     - Publishes a ``GameVictoryEvent``.

Level Manager
-------------

The game uses a **level manager** script that controls progression between levels:

**Campaign Mode** (``level_manager.lua``):

.. code-block:: lua

   levels = {
       { script = "level1.lua", name = "Asteroid Field",  boss = "Mechanical Parasite" },
       { script = "level2.lua", name = "Enemy Territory", boss = "Bio-Guardian" },
       { script = "level3.lua", name = "Alien Hive",      boss = "Hive Totem" },
       { script = "level4.lua", name = "Final Assault",   boss = "Leviathan Warship" }
   }

Each level script defines timed waves, boss encounters, powerup drops, and wall obstacles. When the boss is defeated, ``notify_level_completed()`` is called, which triggers the level manager to load the next level.

**Infinite Mode** (``level_manager_infinite.lua``):

Loads ``level0.lua`` directly. No boss, no progression. Enemies spawn in endless waves with increasing difficulty. The game ends when all players are dead.

.. code-block:: lua
   :caption: level0.lua (excerpt)

   function update(deltaTime)
       game_time = game_time + deltaTime

       if game_time >= next_wave_time then
           spawn_wave()
           next_wave_time = game_time + current_interval
       end
   end

The infinite mode scales difficulty every 3 waves: more enemies, faster spawns, harder enemy types, wall obstacles, and bonus elite squads.

---

3. Game Systems
===============

Systems contain all game logic. They are C++ classes implementing ``ISystem::update(Registry&, float)``.

Server Systems
--------------

.. list-table::
   :widths: 25 75
   :header-rows: 1

   * - System
     - Description
   * - ``GameRulesSystem``
     - Executes Lua scripts. Bridges C++ functions to Lua. Manages level progression.
   * - ``AISystem``
     - Controls enemy behavior based on the ``AI_enemy`` component: straight movement, sinusoidal patterns, shooting, kamikaze diving.
   * - ``ForcePodSystem``
     - Handles Force Pod attachment to players and synchronized shooting. When a player shoots, the pod fires additional bullets based on its type (base, red/spread, blue/ricochet, yellow/laser).
   * - ``WeaponSystem``
     - Reads ``ShootEvent`` from the EventBus and creates bullet entities using the entity factory.
   * - ``MovementSystem``
     - Updates entity positions: ``position += velocity * deltaTime``. Marks modified entities as dirty for network sync.
   * - ``CollisionSystem``
     - AABB collision detection. Handles damage application, powerup collection, Force Pod upgrades, and entity destruction.
   * - ``StayWSystem``
     - Clamps player entities within screen bounds. Destroys bullets and enemies that go off-screen.
   * - ``ServerNetworkSystem``
     - Broadcasts compressed delta snapshots to all clients. Resets player velocities after each tick. Manages client connections and ACKs.

---

4. Game Server Assembly
=======================

The ``GameServer`` class creates an ``Engine`` instance, adds all systems, and configures the network layer with game-specific callbacks.

Configuring the Network System
------------------------------

**Player Factory**: Tells the network system what to do when a new client connects.

.. code-block:: cpp

   _networkSystem->setPlayerFactory([&](Registry& reg, uint8_t shipColor) {
       if (shipColor == 0)
           return _factory.createEntity("player", reg, randomX, randomY);
       else if (shipColor == 1)
           return _factory.createEntity("playergreen", reg, randomX, randomY);
       // ... (4 ship colors supported)
   });

**Input Processor**: Tells the network system how to interpret client packets.

.. code-block:: cpp

   _networkSystem->setInputProcessor(
       [&](Registry& reg, Entity id, const std::vector<char>& data) {
           _gameManage->handleIncomingPacket(reg, id, data);
       });

System Assembly Order
---------------------

.. code-block:: cpp
   :caption: GameServer::init()

   _engine.addSystem(std::move(gameRulesSystem));   // 1. Lua: spawn enemies
   _engine.addSystem(std::make_unique<AISystem>());  // 2. Enemy AI + shoot
   _engine.addSystem(std::make_unique<ForcePodSystem>());  // 3. Pod shoot
   _engine.addSystem(std::make_unique<WeaponSystem>());    // 4. Create bullets
   _engine.addSystem(std::make_unique<MovementSystem>());  // 5. Move entities
   _engine.addSystem(std::make_unique<CollisionSystem>()); // 6. Detect hits
   _engine.addSystem(std::make_unique<StayWSystem>());     // 7. Bounds check
   _engine.addSystem(std::move(_networkSystem));           // 8. Broadcast state

Running the Engine
------------------

Each ``GameServer`` runs in its own thread, started by the Lobby:

.. code-block:: cpp

   void GameServer::start() {
       _running = true;
       _gameThread = std::thread(&GameServer::run, this);
   }

   void GameServer::run() {
       init();
       _engine.run();  // Blocks until _engine.stop() is called
   }