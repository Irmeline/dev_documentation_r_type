===============================
Creating a Game with the Engine
===============================

This guide explains how to use the core Engine to create a new game, using R-Type as an example. It covers the data-driven workflow, from defining entities in JSON to scripting level behavior with Lua.

The fundamental principle of the engine is **decoupling data from code**. Most of the game's logic and content is defined in external configuration files, not in C++.

Workflow Overview
-----------------

Creating a game like R-Type involves four main steps:

1.  **Define Entity "Blueprints"**: Create JSON files that describe the components of each type of entity.
2.  **Script Level Behavior**: Create Lua scripts that define the sequence of events in a level.
3.  **Implement Game Systems**: Write C++ classes (`ISystem`) that contain the core game logic (collisions, AI, etc.).
4.  **Assemble and Run the Engine**: Write a main class (`GameServer`) that creates an `Engine` instance, adds all the necessary systems to it, and launches the main game loop.

---

1. Entity Configuration (JSON Blueprints)
=========================================

Every type of entity in the game is defined as a "template" in a ``.json`` file. The `ServerEntityFactory` reads these files at startup.

Location
--------
``assets/templates/``
or
The folder that you wanted because you just need to precise the folder of your configuration files in the server code
Structure
---------
Each JSON file contains a ``components`` object. Inside, each key is the name of a Component, and its value is an object containing the initial data for that component.

Example: `bydo_scout.json`
-----------------------------
This file defines a basic enemy.

.. code-block:: json
   :caption: assets/templates/bydo_scout.json

   {
     "components": {
       "Health": {
         "hp": 10
       },
       "Hitbox": {
         "width": 66,
         "height": 66
       },
       "Velocity": {
         "dx": -200.0,
         "dy": 0
       },
       "AI_enemy": {
         "behavior": "STRAIGHT_SHOOTER",
         "scoreValue": 100,
         "moveSpeed": 200,
         "shootPeriodTicks": 120
       }
     }
   }

- **`Health`, `Hitbox`, `Velocity`**: These are basic shared components.
- **`AI_enemy`**: This is a game-specific component. The ``"behavior"`` key tells the `AISystem` which logic to apply to this entity. The other fields are parameters for that behavior.

Example: `player_bullet.json`
--------------------------------
This defines a projectile. Note that it has a `Damage` component but no `Health`.

.. code-block:: json
   :caption: assets/templates/player_bullet.json

   {
     "components": {
       "Velocity": { "dx": 1000.0, "dy": 0 },
       "Hitbox": { "width": 32, "height": 8 },
       "Damage": { "value": 10 },
       "Bullet": {}
     }
   }

To create a new type of enemy, you simply need to create a new JSON file and define its starting components.

---

2. Level Scripting (Lua)
========================

The flow of each level is controlled by a Lua script. The `GameRulesSystem` is responsible for loading and executing this script.

Location
--------
``assets/levels/``

How It Works
------------
The C++ `GameRulesSystem` creates a Lua virtual machine and "binds" several C++ functions, making them available to be called from Lua. The script must define a global ``update(deltaTime)`` function, which the `GameRulesSystem` calls at every game tick.

Key Bound Functions
-------------------
The engine exposes several crucial functions to Lua:

- ``create_entity(templateName, x, y)``: Asks the `ServerEntityFactory` to create a new entity from a JSON blueprint at a specific position.
- ``get_random_position(min, max)``: A utility function to help with randomized spawning.
- ``get_player_score(playerId)``: Retrieves the current score of a specific player.
- ``get_all_player_ids()``: Returns a list of all currently connected players.

Example: `test_level.lua`
-------------------------

.. code-block:: lua
   :caption: assets/levels/test_level.lua

   -- State variables for this level script
   game_time = 0.0
   enemy_spawn_timer = 3.0

   -- This function is called 60 times per second by the C++ Engine
   function update(deltaTime)
       game_time = game_time + deltaTime
       enemy_spawn_timer = enemy_spawn_timer - deltaTime

       -- If the timer is finished...
       if enemy_spawn_timer <= 0 then
           print("[LUA] Spawning an enemy wave.")
           
           -- Get the number of connected players
           local num_players = #get_all_player_ids()
           
           -- Spawn one enemy for each player at a random height
           for i = 1, num_players do
               local random_y = get_random_position(100.0, 980.0)
               create_entity("bydo_scout", 2000, random_y)
           end
           
           -- Reset the timer for the next wave
           enemy_spawn_timer = 5.0 
       end
   end

To create a new level, you simply need to write a new ``.lua`` file with its own unique logic in the ``update`` function.

---

3. The Engine: Assembling and Running Systems
==============================================

The ``Engine`` class is the heart of a game instance. It doesn't have any game logic itself; it is an orchestrator that holds a ``Registry`` and executes a list of **Systems** in a continuous loop.

Adding Systems
--------------
Systems contain all the game's logic. You must create and add them to the Engine instance for them to have any effect. This is typically done in the constructor of your main game class (e.g., `GameServer`).

The **order** in which you add systems is **critical**, as it defines the order of execution within each game tick. A logical order is essential for a stable simulation.

.. code-block:: cpp
   :caption: GameServer.cpp (constructor - System Assembly)

   // 1. First, add systems that handle inputs and game rules.
   _engine.addSystem(_networkSystem); // Contains the InputProcessor
   _engine.addSystem(std::make_unique<GameRulesSystem>(...));
   _engine.addSystem(std::make_unique<AISystem>(...));

   // 2. Then, add systems that trigger actions based on those rules.
   _engine.addSystem(std::make_unique<WeaponSystem>(...));

   // 3. Next, simulate the physics based on the new state.
   _engine.addSystem(std::make_unique<MovementSystem>());
   _engine.addSystem(std::make_unique<CollisionSystem>(...));

   // 4. Finally, add cleanup systems.
   _engine.addSystem(std::make_unique<JanitorSystem>(...));

   // The NetworkSystem's `update` will be called again at the end of the tick
   // (in this list) to broadcast the final state.

Running the Engine
------------------
Once all systems are added, the game is ready to be launched. The `Engine` provides a ``run()`` method that starts the main game loop. Since a game server must run in parallel to the lobby, this is typically done in a separate thread.

.. code-block:: cpp
   :caption: GameServer.cpp (start method)

   void GameServer::start() {
       _isRunning = true;
       // We create a new thread that will execute our game loop function.
       _gameThread = std::thread(&GameServer::gameLoop, this);
   }

   void GameServer::gameLoop() {
       std::cout << "[GameServer] The game loop is starting." << std::endl;

       // We call the Engine's run method, giving it a function
       // that it will check at each tick to know if it should continue.
       _engine.run([this]() { 
           return _isRunning.load(); 
       });

       std::cout << "[GameServer] The game loop has ended." << std::endl;
   }

The `Engine::run` method itself contains the time-managed loop that calls `system->update()` 60 times per second, providing the crucial `deltaTime` for physics and timers.

.. code-block:: cpp
   :caption: Engine::run (conceptual)

   void Engine::run(std::function<bool()> condition) {
       auto lastTime = std::chrono::steady_clock::now();
       
       while (condition()) {
           auto currentTime = std::chrono::steady_clock::now();
           float deltaTime = std::chrono::duration<float>(currentTime - lastTime).count();
           lastTime = currentTime;

           // Execute all registered systems in order.
           for (const auto& system : m_systems) {
               system->update(registry, deltaTime);
           }

           // (Loop timing logic to maintain a fixed tick rate)
       }
   }

---

4. Game Server Assembly & Configuration
=======================================
*(Cette section remplace l'ancienne "Game Server Assembly")*

The `GameServer` class is the main entry point for a game instance. It is responsible for creating an `Engine`, adding all the systems, and configuring the generic network layer with game-specific logic.

Configuring the Network System
------------------------------
The `GenericServerNetworkSystem` is part of the Engine. It's a "blank slate" that needs to be told how your game works. This is done by providing it with callback functions (lambdas).

**`setPlayerFactory`**
  This tells the network system what to do when a brand-new client connects. The provided function is responsible for creating the player's entity.

  .. code-block:: cpp
     :caption: GameServer.cpp (constructor)

     _networkSystem->setPlayerFactory(
         // `[&]` captures `this` to access the factory
         [&](Registry& reg, const udp::endpoint& ep) {
             // We use our game's entity factory to create a player from the JSON template
             return _factory.createEntity("player", reg, 100.0, 540.0);
         }
     );

**`setInputProcessor`**
  This tells the network system how to interpret the raw data packets sent by clients.

  .. code-block:: cpp
     :caption: GameServer.cpp (constructor)

     _networkSystem->setInputProcessor(
         // `[&]` captures `this` to access the EventBus
         [&](Registry& reg, Entity playerId, const std::vector<char>& data) {
             // We know our game uses PlayerInputPacket
             PlayerInputPacket packet;
             std::memcpy(&packet, data.data(), sizeof(packet));

             // We translate the input into events for other systems to handle.
             if (packet.input_mask & IN_SHOOT) {
                 _eventBus.publish(PlayerWantsToShootEvent{playerId});
             }
             // (Logic to parse movement and update Velocity component...)
         }
     );

By providing these functions, you "teach" the generic network engine the rules of your specific game, allowing it to function correctly without containing any hardcoded game logic itself.