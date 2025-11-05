Overview
========

The R-Type project is divided into **four distinct parts**:

1. **Engine (Core Library)**
   - A **static library** containing the generic heart of the project.
   - Implements the Entity Component System (`Registry`, `sparse_array`), the `Engine` loop, and the `EventBus`.
   - Contains **abstract interfaces** for graphics, audio, and input (`IGraphicsFactory`, `IRenderWindow`, `ISprite`...).

2. **Graphics/Audio Backends (Plugin Libraries)**
   - **Shared libraries** (`.so`/`.dll`/`.dylib`) that act as "translators" for a specific multimedia library.
   - Example: ``SFMLBackend`` implements the engine's abstract interfaces using SFML.
   - The engine can load any compatible backend at runtime, making it truly cross-platform and flexible.

3. **Game Logic (R-Type Specific)**
   - Contains all code specific to the R-Type game.
   - **Server Logic:** `AISystem`, `CollisionSystem`, `GameRulesSystem` (with Lua scripting), `WeaponSystem`.
   - **Client Logic:** `ClientNetworkSyncSystems`, `ScrollingSystem`.
   - **Shared Logic:** Common components (`Position`, `Health`...) and network protocol.

4. **Executables (Client & Server)**
   - Thin applications that assemble the Engine, load a backend (for the client), and add the R-Type specific game systems.
