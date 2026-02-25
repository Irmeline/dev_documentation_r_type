============================
Creating a Game Client
============================

This guide explains how to build the client-side application. The client connects to the server, renders the game world received from the network, and sends player inputs.

A key feature of the engine is its **graphics abstraction layer**, which allows the game to be rendered with different graphics libraries by loading a different backend plugin at runtime.

Client Workflow Overview
------------------------

1. **Load Backend & Initialize Graphics**: Dynamically load a graphics backend (e.g., ``SFMLBackend.so``) and use it to create a window.
2. **Define Visual Blueprints**: Create JSON files that describe the visual properties of each entity.
3. **Connect to Lobby**: Communicate with the server's lobby to join or create a game, receiving a dedicated game port.
4. **Assemble Client Systems**: Create an ``Engine`` instance and add all necessary client-side systems.
5. **Run the Session Loop**: Menu → Game → Cleanup → Menu (supports returning to menu after game over or victory).

---

1. Graphics Abstraction & Backend Plugins
=========================================

The engine is completely decoupled from any specific graphics library through abstract interfaces and a plugin system.

Key Interfaces
--------------

- ``IGraphicsFactory``: Creates all graphics/audio objects (windows, sprites, textures, fonts, sounds).
- ``IRenderWindow``: Represents the game window. Handles event polling, key state, and provides access to the renderer.
- ``IRenderer``: Handles draw calls for sprites, text, and shapes (rectangles for UI).
- ``ISprite``, ``ITexture``, ``IFont``, ``IText``, ``ISound``, ``ISoundBuffer``: Abstract representations of drawable and audible objects.

Loading the Plugin
------------------

.. code-block:: cpp

   PluginLoader plugin("./libSFMLBackend.so");
   std::unique_ptr<IGraphicsFactory> factory = plugin.createFactory();
   auto window = factory->createWindow(1920, 1080, "R-Type Client");

To switch to a different library, implement the interfaces in a new shared library and change the path. No engine or game code changes needed.

---

2. Client Configuration (JSON Files)
====================================

``resources.json``
------------------
A catalogue of all art assets. Maps resource IDs to file paths. The ``ResourceManager`` loads these at startup.

.. code-block:: json

   {
     "textures": [
       { "id": "player_ship", "path": "assets/sprites/blue_ship.png" },
       { "id": "enemy1",      "path": "assets/sprites/enemy1.png" }
     ],
     "fonts": [
       { "id": "main_font", "path": "assets/font/PressStart2P-Regular.ttf" }
     ],
     "sounds": [
       { "id": "player_shoot_sound", "path": "assets/audio/player_shoot_sound.wav" }
     ]
   }

Entity Templates
----------------
Each entity type has a JSON file defining its visual representation. The ``ClientEntityFactory`` uses these to create sprites and animations when the server notifies of entity creation.

.. code-block:: json
   :caption: config_files/enemy1.json

   {
     "sprite": {
       "texture": "enemy1",
       "rect": { "left": 0, "top": 0, "width": 33, "height": 34 },
       "scale": { "x": 2.0, "y": 2.0 }
     },
     "animation": {
       "frameWidth": 33,
       "frameHeight": 34,
       "frameCount": 3,
       "frameDuration": 0.15
     }
   }

---

3. The Dual Registry System
===========================

To prevent conflicts between server-controlled entities and client-only entities, the client uses **two separate** ``Registry`` instances:

``gameRegistry`` (from ``clientEngine.getRegistry()``)
  Contains all entities synchronized with the server (players, enemies, bullets). Entity IDs are **dictated by the server** via snapshot packets.

``localRegistry``
  Contains client-only entities (scrolling backgrounds, UI text elements). The client manages its own IDs using ``spawn_entity()``.

The ``RenderSystem`` draws both registries. Local entities (backgrounds) are drawn first, then game entities, then UI texts from the local registry on top.

---

4. Client Systems
=================

.. list-table::
   :widths: 20 80
   :header-rows: 1

   * - System
     - Description
   * - ``InputSystem``
     - Reads keyboard state and key events. Movement keys (Up/Down/Left/Right) are read continuously. Shoot (Space) is read as a single event per press. Publishes ``LocalPlayerInputEvent`` with the combined input bitmask.
   * - ``UISystem``
     - Manages all in-game UI. Reads events from the EventBus (``PlayerStateUIEvent``, ``DEADEvent``, ``LevelCompleteUIEvent``, ``VictoryUIEvent``). Has 4 states: GAMEPLAY (HUD visible), LEVEL_COMPLETE (auto-dismiss overlay), DEATH (game over screen), VICTORY (leaderboard). Handles ENTER key to trigger return to menu.
   * - ``ClientNetworkSystem``
     - Sends input packets to the server. Receives compressed snapshots asynchronously. Decompresses and parses them. Applies destructions, creations, and position updates to the game registry. Publishes UI events based on game state flags and player states. Sends ACK packets for reliable delivery.
   * - ``ScrollingSystem``
     - Moves background entities leftward and wraps them to create infinite scrolling.
   * - ``AnimationSystem``
     - Advances sprite sheet animations by updating texture rectangles based on elapsed time.
   * - ``RenderSystem``
     - Draws all entities in order: local sprites (backgrounds) → game sprites → game texts → **UI texts from local registry** (drawn last, always on top).

System Execution Order
----------------------

.. code-block:: text

   1. InputSystem     → reads keys, publishes input event
   2. UISystem        → reads UI events from previous frame, updates HUD/overlays
   3. NetworkSystem   → sends inputs, receives snapshots, clears all events
   4. ScrollingSystem → scrolls backgrounds
   5. AnimationSystem → advances sprite frames
   6. RenderSystem    → draws everything (LAST)

The UISystem runs **before** the NetworkSystem because the network system calls ``eventBus.clearAllEvents()`` at the end of its update. UI events published asynchronously (from network receive callbacks) are consumed by the UISystem in the next frame before they are cleared.

---

5. The Session Loop
===================

The client's ``main.cpp`` implements a **session loop** that supports returning to the menu after a game ends.

.. code-block:: text

   while (window is open) {

       PHASE 1: MENU
           Create fresh Engine, registries, MenuSystem
           Show menu UI (create room, join room, infinite mode)
           Wait for game to start

       PHASE 2: GAME
           clientEngine.run([&]() {
               if (uiSystem->shouldReturnToMenu())
                   return false;
               return window->isOpen();
           });

       PHASE 3: CLEANUP
           io_context.stop()
           networkThread.join()
           clientEngine.clearSystems()
           → All local variables destroyed at end of scope
           → Loop back to PHASE 1 with fresh state
   }

Each session creates **entirely new** objects (Engine, registries, MenuSystem, network context). No residual state between sessions.

Return to Menu Flow
-------------------

.. code-block:: text

   Game Over / Victory screen displayed
       → Player presses ENTER
       → UISystem sets _returnToMenu = true
       → clientEngine.run() lambda returns false
       → Game loop stops
       → Network thread joined
       → Systems cleared, scoped objects destroyed
       → Outer while loop creates fresh menu
       → Player can start a new game

---

6. The UI System
================

The ``UISystem`` manages all in-game UI using text entities in the local registry. It creates text entities via the ``IGraphicsFactory`` and the ``ResourceManager`` (font: ``main_font``).

UI States
---------

.. list-table::
   :widths: 20 80
   :header-rows: 1

   * - State
     - Display
   * - ``GAMEPLAY``
     - HUD visible: HP bar (color changes green→yellow→red), current score, current level number.
   * - ``LEVEL_COMPLETE``
     - Overlay showing "LEVEL X COMPLETE!" with scores ranked by player. Auto-dismisses after 4 seconds.
   * - ``DEATH``
     - "GAME OVER" overlay with final score. "Press ENTER to return to menu" prompt. HUD hidden.
   * - ``VICTORY``
     - "VICTORY!" overlay with total score, leaderboard ranking all players. "Press ENTER to return to menu" prompt. HUD hidden.

UI Events
---------

These events are published by the ``ClientNetworkSystem`` when it receives snapshot data:

- ``PlayerStateUIEvent { hp, maxHp, score }`` — updates the HUD each frame.
- ``LevelCompleteUIEvent { levelNumber, playerScores }`` — triggers the level complete overlay.
- ``GameOverUIEvent { finalScore }`` — triggers the game over screen.
- ``VictoryUIEvent { totalScore, leaderboard }`` — triggers the victory screen.
- ``DEADEvent { entityId, isPlayer, scoreValue }`` — triggers game over when the local player is destroyed.

---

7. The Menu System
==================

The ``MenuSystem`` handles pre-game UI: main menu, room creation, room browser, infinite mode selection. It communicates with the lobby via UDP, processes room list updates, and manages the lobby connection.

The ``MenuRenderSystem`` draws the menu entities (buttons, text, panels) from the ``menuRegistry`` using the ``IRenderer``'s shape drawing capabilities (``drawFilledRect``, ``drawRect``).

When the player selects a game mode and the lobby confirms, the menu system sets ``shouldStartGame() = true``, which triggers the main loop to initialize the game systems and transition to the game phase.