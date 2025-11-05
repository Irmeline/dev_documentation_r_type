============================
Creating a Game Client
============================

This guide explains how to build the client-side application for a game using the Engine. The client's primary roles are to connect to the server, render the game world received from the network, and send player inputs.

A key feature of the engine is its **graphics abstraction layer**, which allows the game to be rendered with different graphics libraries (like SFML or SDL) by simply loading a different backend plugin at runtime.

Client Workflow Overview
------------------------
1.  **Load Backend & Initialize Graphics**: Dynamically load a graphics backend (e.g., ``SFMLBackend.so``) and use it to create a window.
2.  **Define Visual "Blueprints"**: Create JSON files that describe the visual and audio properties of each entity.
3.  **Connect to Lobby**: Communicate with the server's lobby to join or create a game, receiving a dedicated game port in return.
4.  **Assemble Client Systems**: Create an `Engine` instance and add all necessary client-side systems (`Input`, `Network`, `Render`, `Animation`, etc.).
5.  **Run the Main Loop**: Start the engine's loop, which will drive all systems to update the game state and render it to the window.

---

1. Graphics Abstraction & Backend Plugins
=========================================

The engine is completely decoupled from any specific graphics library. This is achieved through a set of **abstract interfaces** and a **plugin system**.

Key Interfaces (in `Engine/include/Graphics/`)
----------------------------------------------
- ``IFactory``: Creates all graphics/audio objects (windows, sprites, textures...).
- ``IRenderWindow``: Represents the game window.
- ``IRenderer``: Handles all draw calls.
- ``ISprite``, ``ITexture``, ``IFont``, ``ISound``, etc.: Abstract representations of resources and drawable objects.

Changing the Graphics Library
-----------------------------
To switch from SFML to another library (e.g., SDL), a developer only needs to:
1.  Implement a new backend by creating classes like `SDLFactory`, `SDLWindow`, `SDLSprite` that inherit from the engine's interfaces.
2.  Compile this new backend as a shared library (e.g., ``SDLBackend.so``).
3.  Change the path in the client's `main.cpp` to load the new plugin file.

**No changes are needed in the Engine or the core client logic.**

Loading the Plugin
------------------
The `PluginLoader` class handles the dynamic loading of the backend at runtime.

.. code-block:: cpp
   :caption: main.cpp (Client) - Loading the Backend

   #include "Engine/Core/PluginLoader.hpp"

   // Define the path to the backend library file
   const std::string PLUGIN_PATH = "./lib/libSFMLBackend.so";
   
   // Load the plugin
   PluginLoader plugin(PLUGIN_PATH);

   // Create the factory. `factory` is a unique_ptr<IFactory>.
   // The code from this point on is generic.
   std::unique_ptr<IFactory> factory = plugin.createFactory();

   // Use the generic factory to create a window.
   auto window = factory->createWindow(1920, 1080, "R-Type Client");

---

2. Client Configuration (JSON Files)
====================================

The client uses its own set of JSON files to define the "look and feel" of the game.

Location
--------
``assets/client_config/``
or
The folder that you wanted because you just need to precise the folder of your configuration files in the server code
Structure

`resources.json`
----------------
This file acts as a **catalogue of all art assets**. It maps a simple resource ID to a file path. The `ResourceManager` loads all these assets into memory at startup.

.. code-block:: json
   :caption: assets/client_config/resources.json

   {
     "textures": {
       "player_ship": "assets/sprites/player_ship.png",
       "bydo_scout": "assets/sprites/bydo_scout_sheet.png",
       "background_space": "assets/backgrounds/space.png"
     },
     "fonts": {
       "main_font": "assets/fonts/PressStart2P.ttf"
     },
     "sounds": {
       "player_shoot": "assets/sfx/shoot.wav",
       "enemy_explosion": "assets/sfx/explosion.wav"
     }
   }

`client_templates.json`
-----------------------
This file is the **visual blueprint** for entities. The `ClientEntityFactory` uses it to "dress up" entities created by the server. It maps a `templateName` (received from the server) to a set of presentation components.

.. code-block:: json
   :caption: assets/client_config/client_templates.json

   {
     "bydo_scout": {
       "sprite": {
         "resourceId": "bydo_scout",
         "scale": { "x": 1.5, "y": 1.5 }
       },
       "animation": {
         "frameWidth": 33,
         "frameHeight": 34,
         "frameCount": 3,
         "frameDuration": 0.15
       },
       "sounds": {
         "on_death": "enemy_explosion"
       }
     }
   }

---

3. The Dual Registry System
===========================

To prevent conflicts between server-controlled entities and client-only entities, the client uses **two separate `Registry` instances**.

`networkRegistry`
  Contains all entities synchronized with the server (players, enemies, projectiles). Entity IDs in this registry are **dictated by the server**.

`localRegistry`
  Contains all entities that exist only on the client (backgrounds, UI elements, visual effects). The client generates its own IDs for this registry using ``spawn_entity()``.

This separation is managed in the client's `main.cpp`.

---

4. Assembling and Running the Client
=====================================

The client's `main.cpp` is the master assembler. It creates all the services and systems and injects their dependencies.

System Assembly
---------------
The order in which systems are added to the `Engine` is crucial.

.. code-block:: cpp
   :caption: main.cpp (Client) - System Assembly

   // The Client's main function
   int main() {
       // ... (Initialisation of factories, managers, window...)

       // Create the two registries
       Registry networkRegistry;
       Registry localRegistry;

       // Create systems, passing the correct registry
       clientEngine.addSystem(std::make_unique<InputSystem>(*window, eventBus));
       
       // NetworkSystem only works on the networkRegistry
       auto networkSystem = std::make_unique<ClientNetworkSystem>(io_context, networkRegistry, ...);
       clientEngine.addSystem(std::move(networkSystem));
       
       // ScrollingSystem only works on the localRegistry
       clientEngine.addSystem(std::make_unique<ScrollingSystem>(localRegistry, SCREEN_WIDTH));
       
       // Animation and Audio systems need to work on both registries
       clientEngine.addSystem(std::make_unique<AnimationSystem>());
       clientEngine.addSystem(std::make_unique<AudioSystem>(...));
       
       // RenderSystem needs to draw both registries in the correct order
       clientEngine.addSystem(std::make_unique<RenderSystem>(*window, networkRegistry, localRegistry));

       // ... (Launch the engine loop)
   }

Running the Engine
------------------
The client uses a manual game loop controlled by the window's state. The `Engine`'s `run` method is provided with a lambda function that checks if the window is still open.

.. code-block:: cpp
   :caption: main.cpp (Client) - Main Loop

   // Launch the network thread in the background
   std::thread networkThread([&]() { io_context.run(); });
        
   // Launch the engine's main loop
   clientEngine.run([&]() {
       // This code is executed at the start of each frame
       eventBus.clearAllEvents();
       return window->isOpen();
   });

   // ... (Cleanup code to stop the network thread)