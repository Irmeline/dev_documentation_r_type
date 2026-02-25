Overview
========

The R-Type project is divided into **four distinct parts**:

1. **Engine (Core Library)**
   A **static library** containing the generic heart of the project.
   It implements the Entity Component System (``Registry``, ``sparse_array``), the ``Engine`` loop, and the ``EventBus``.
   It contains **abstract interfaces** for graphics, audio, and input (``IGraphicsFactory``, ``IRenderWindow``, ``ISprite``, etc.).
   It also provides the generic ``ServerNetworkSystem`` and ``ClientNetworkSystem`` that handle snapshot broadcasting, compression, and reliable delivery.

2. **Graphics/Audio Backends (Plugin Libraries)**
   **Shared libraries** (``.so``/``.dll``/``.dylib``) that act as translators for a specific multimedia library.
   Example: ``SFMLBackend`` implements the engine's abstract interfaces using SFML.
   The engine can load any compatible backend at runtime via the ``PluginLoader``, making it truly cross-platform and flexible.

3. **Game Logic (R-Type Specific)**
   Contains all code specific to the R-Type game.

   - **Server Logic:** ``AISystem``, ``CollisionSystem``, ``GameRulesSystem`` (with Lua scripting), ``WeaponSystem``, ``ForcePodSystem``, ``StayWSystem``.
   - **Client Logic:** ``ClientNetworkSyncSystems``, ``ScrollingSystem``, ``AnimationSystem``, ``UISystem``.
   - **Shared Logic:** Common components (``Position``, ``Health``, ``Velocity``, etc.) and the binary network protocol.
   - **Lobby:** Room management, multiplayer matchmaking, infinite mode, kick/ban, admin console.

4. **Executables (Client & Server)**
   Thin applications that assemble the Engine, load a backend (for the client), and add the R-Type specific game systems.

   - The **Server** executable starts a Lobby that listens on a configurable port, spawns ``GameServer`` instances per room, and provides an admin dashboard.
   - The **Client** executable loads the SFML plugin, connects to the lobby, and runs a session loop that supports returning to menu after game over or victory.