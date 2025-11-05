=============================
Comparative & Technical Study
=============================

This document outlines the key technical decisions made during the development of the engine and the R-Type game. Each choice is justified by comparing it against viable alternatives, highlighting the trade-offs considered and explaining why the selected technology or pattern was the best fit for the project's goals: **performance, modularity, and rapid development**.

---

1. Core Language: C++20
=======================

**Selected: C++20**

- **Justification**: C++20 provides a powerful combination of high-level abstractions and low-level memory control, which is essential for building a high-performance game engine. Features like templates, smart pointers (`std::unique_ptr`, `std::shared_ptr`), and compile-time evaluation (`constexpr`) are fundamental to the ECS architecture. C++20 specifically adds valuable features like concepts and modules, pushing the language forward. Its vast ecosystem and direct hardware access make it the industry standard for engine development.

- **Alternatives Considered**:
  - **Rust**: Offers superior memory safety guarantees ("fearless concurrency") which eliminates entire classes of bugs (dangling pointers, data races). However, its learning curve is steeper, its ecosystem for game development is less mature than C++'s, and compilation times can be longer. For this project, the proven performance and vast library support of C++ were deemed more critical than Rust's safety guarantees.
  - **C# (with Unity/Godot)**: C# is a high-level language that greatly simplifies development. Game engines like Unity provide a complete, feature-rich environment out of the box. However, the goal of this project was not just to make a game, but to **build an engine**. Using a pre-made engine would have defeated the primary educational and architectural purpose. C# without a major engine (e.g., with .NET) lacks the same level of performance-critical libraries as C++.

**Conclusion**: C++20 was the optimal choice, providing the necessary performance and control to build an engine from the ground up, in line with the project's core objectives.

---

2. Architectural Pattern: Entity Component System (ECS)
========================================================

**Selected: Entity Component System (ECS)**

- **Justification**: ECS was chosen over traditional Object-Oriented Programming (OOP) for its superior data locality and flexibility.
- **Decoupling**: It strictly separates data (Components) from logic (Systems). This prevents the creation of monolithic "god objects" and makes code easier to maintain, test, and reason about.
- **Performance**: By storing components of the same type contiguously in memory (in our `sparse_array`), systems can iterate over data very quickly, leading to better cache utilization (Data-Oriented Design). This is critical for performance in a game with many entities.
- **Flexibility**: Entities are defined by composition, not inheritance. A new type of enemy can be created by simply assembling a different set of components in a JSON file, without writing any new C++ classes.

- **Alternatives Considered**:
- **Traditional OOP (Inheritance-based)**: A more classical approach would involve a `GameObject` base class with derived classes like `Player`, `Enemy`, and `Bullet`. This quickly leads to problems:
- **The Diamond Problem**: Multiple inheritance can become complex (e.g., an `Enemy` that is also a `FlyingObject` and a `ShootingObject`).
- **Inflexibility**: If you want an enemy that can sometimes have a shield, do you create a new `ShieldedEnemy` class? What if it can also drop items? The class hierarchy explodes.
- **Poor Performance**: Data for a single object is scattered in memory, leading to frequent cache misses when iterating over many objects.

**Conclusion**: ECS provides a far more scalable, performant, and flexible architecture for a game engine, making it the clear choice.

---

3. Graphics & Audio: Abstraction Layer + Backend Plugins
========================================================

**Selected: Abstract Interface Layer with Dynamic Backend Loading**

- **Justification**: The engine's core is completely independent of any specific multimedia library. We defined a set of abstract interfaces (`IFactory`, `IRenderWindow`, `ISprite`, etc.). A concrete implementation (e.g., `SFMLBackend`) is compiled as a shared library (`.so`/`.dll`) and loaded at runtime by the `PluginLoader`.
- **Modularity**: This is the ultimate form of decoupling. The engine can be rendered with SFML, SDL, or any other library by simply creating a new backend plugin and changing a single line in the `main` configuration. The engine code itself remains untouched.
- **Clear Separation**: The responsibilities are perfectly divided. The Engine defines *what* should be done (e.g., "draw a sprite"), while the Backend defines *how* it should be done (e.g., "call `sf::RenderWindow::draw`").

- **Alternatives Considered**:
- **Direct SFML Integration**: The simplest approach would be to use SFML types (`sf::Sprite`, `sf::Texture`) directly within the engine's components and systems.
- **Pros**: Much faster to implement initially.
- **Cons**: The engine would be permanently "married" to SFML. Porting to another library (e.g., for a console release, or if SFML development were to cease) would require rewriting large parts of the engine. This fails the "reusable engine" goal.
- **Compile-Time Abstraction (#ifdef)**: Using preprocessor directives (`#ifdef USE_SFML ... #elif USE_SDL ...`) to switch between libraries.
- **Cons**: Leads to messy, hard-to-read code. It requires recompiling the entire engine to switch backends and makes it difficult to support both simultaneously.

**Conclusion**: The plugin-based architecture, while the most complex to set up, provides unparalleled flexibility and is the hallmark of a professionally designed, reusable game engine.

---

4. Networking: ASIO with Custom UDP Protocol
============================================

**Selected: ASIO (Standalone) with a Custom Binary UDP Protocol**

- **Justification**:
- **ASIO**: Chosen for its high performance, low-level control, and asynchronous, non-blocking model. It allows the server to handle network I/O in a dedicated thread without ever blocking the main game loop, which is essential for a real-time server. Its standalone nature makes it lightweight and portable.
- **UDP**: The User Datagram Protocol was chosen for in-game communication. For a fast-paced action game, the low latency of UDP is far more important than the guaranteed delivery of TCP. If a packet containing an old position is lost, it's better to simply ignore it and use the next, more recent packet.
- **Custom Binary Protocol**: We designed a "Snapshot of Changes" protocol. This is highly efficient as it only sends deltas (creations, destructions, modifications) rather than the entire world state at every tick, saving significant bandwidth. A binary format is much more compact and faster to parse than a text-based format like JSON or XML.

- **Alternatives Considered**:
- **TCP**: Provides reliable, ordered delivery.
- **Cons**: Unsuitable for real-time action. A single lost packet can cause the entire stream to halt ("head-of-line blocking"), leading to massive lag spikes while the protocol tries to retransmit the lost data.
- **Higher-Level Libraries (e.g., ENet, RakNet)**: These libraries provide a "reliable UDP" layer and other features out of the box.
- **Pros**: Faster to get a reliable connection up and running.
- **Cons**: They add another layer of abstraction and dependency. For the scope of this project, building the core reliability mechanisms (like tick-based packet ordering) ourselves provided a deeper understanding of network programming challenges.
- **Text-Based Protocol (JSON over UDP)**:
- **Cons**: Extremely inefficient. Serializing and parsing text is slow, and the resulting packets are much larger than a compact binary format, leading to higher bandwidth usage and increased latency.

**Conclusion**: The combination of ASIO's asynchronous model, UDP's low latency, and a custom binary protocol provides the optimal balance of performance, control, and efficiency for this real-time game.

---

5. Scripting: Lua with sol2
===========================

**Selected: Lua, bound via the `sol2` library.**

- **Justification**: Game logic, especially level design and enemy behavior, needs to be changed frequently. Recompiling the entire C++ server for every small tweak is slow and inefficient.
- **Lua**: An extremely lightweight, fast, and easily embeddable scripting language. It is the de-facto standard for game scripting for these reasons.
- **sol2**: A modern, header-only C++ library that makes it incredibly simple and safe to "bind" C++ functions and classes to Lua. It allows our `GameRulesSystem` to call Lua functions (`update`) and our Lua scripts to call C++ functions (`create_entity`).
- **Workflow**: This enables a data-driven workflow where game designers can create complex level scenarios and enemy waves by writing simple Lua scripts, without ever touching the C++ codebase.

- **Alternatives Considered**:
- **No Scripting (Hardcoded C++)**:
- **Cons**: Extremely inflexible. Any change to a level's timing or an enemy's spawn position would require a full recompile of the server. This is untenable for iterative game design.
- **Custom JSON Logic**: Using a complex JSON structure to define conditional events.
- **Cons**: Becomes incredibly convoluted very quickly. A real programming language like Lua is far more expressive and powerful for defining logic (`if`, `for`, functions, variables). JSON is for data, not for logic.
- **Other Scripting Languages (Python, JavaScript)**:
- **Cons**: While powerful, they are generally "heavier" to embed than Lua. They have a larger memory footprint and can be slower to initialize and execute in a real-time context. Lua was specifically designed for this kind of integration.

**Conclusion**: The Lua + sol2 combination provides the best balance of performance, ease of integration, and workflow flexibility, enabling true data-driven game design.

---

6. Security Considerations
==========================

While not a primary focus for a university-level project, considering security is a mark of professional software engineering. An online game is exposed to various threats, from cheating to server attacks. Our architectural choices were made with basic security principles in mind.

A. Network Protocol Security
----------------------------

**Selected Approach: Custom Binary UDP Protocol with Server-Side Validation.**

- **Justification**: Our protocol is inherently more secure than text-based alternatives.
- **Obfuscation**: A binary protocol is not human-readable. While it can be reverse-engineered, it presents a higher initial barrier to casual attackers compared to sending easily readable JSON or XML.
- **Minimal Attack Surface**: The server only accepts a few, well-defined packet structures (like `PlayerInputPacket`). Any packet that does not perfectly match the expected size and structure can be immediately discarded, preventing many parsing-based exploits.
- **Server Authority**: **This is the most important security principle.** The client is never trusted. It only sends *intentions* (e.g., "I want to move up"). The server receives this intention, validates it against the game's physics and rules (e.g., "Is the player trying to move through a wall?"), and only then updates the authoritative game state. This prevents a huge range of cheats like speed-hacks or teleportation.

- **Alternatives Considered**:
- **No Validation ("Trust the Client")**: The simplest but most insecure model. A modified client could send packets saying "My position is now (x, y)" or "I just killed the boss", and the server would accept it. This is completely unacceptable for any competitive game.
- **Encryption (e.g., DTLS for UDP)**: Encrypting all UDP traffic would prevent packet sniffing (seeing what other players are doing) and packet injection.
- **Trade-off**: Encryption adds computational overhead and a small amount of latency to every packet. For a fast-paced game, this overhead was deemed unnecessary for the project's scope, but it would be a mandatory consideration for a commercial release, especially for features like player authentication.
- **Anti-Cheat Middleware (e.g., Easy Anti-Cheat, BattlEye)**: These are complex, third-party solutions that perform deep analysis of the client's memory and system to detect cheating software.
- **Trade-off**: Extremely effective but far beyond the scope of this project. They are external services, not an architectural choice.

**Conclusion**: The server-authoritative model is our primary and most effective security layer against cheating, providing a robust foundation without the overhead of full encryption for this project's scale.

B. Server-Side Security
-----------------------

**Selected Approach: Resource Isolation, Input Sanitization, and Rate Limiting.**

- **Justification**: The server must protect itself from being crashed or overloaded by malicious clients.
- **Asynchronous I/O (ASIO)**: Our non-blocking model makes the server inherently more resilient to simple Denial-of-Service (DoS) attacks. A flood of packets from one client will be processed in the I/O thread but will not block the main game loop, preventing the entire simulation from freezing.
- **Input Sanitization**: All data received from clients is treated as untrusted. Our packet processing logic uses `std::memcpy` with fixed sizes (`sizeof(Packet)`). This prevents buffer overflows that could be caused by a client sending an unexpectedly large packet. We immediately discard any packet whose size does not match its expected opcode.
- **(Conceptual) Rate Limiting**: Although not fully implemented, the architecture allows for it. The server could track the number of packets received from each IP address per second. If a client exceeds a reasonable threshold, it can be temporarily ignored or permanently banned, mitigating DoS/DDoS attacks.

- **Alternatives Considered**:
- **Synchronous/Blocking I/O**: A simpler server model where a `recvfrom()` call blocks the entire program until a packet arrives.
- **Cons**: Extremely vulnerable. A single slow or malicious client could freeze the entire server for all other players.

**Conclusion**: By using an asynchronous model and practicing strict input validation, the server is architected to be resilient against common network-based attacks.

C. Data and Scripting Security
------------------------------

**Selected Approach: Sandboxed Lua Environment.**

- **Justification**: Loading external scripts (`.lua`) is a potential security risk if not handled properly. A malicious script could try to access the filesystem, make network calls, or enter an infinite loop.
- **Controlled Environment**: Our `GameRulesSystem` creates a sandboxed Lua state. We use `sol2` to expose only a very limited set of specific C++ functions (`create_entity`, `get_player_score`, etc.). The Lua script **cannot** include arbitrary C++ headers, access pointers, or call dangerous system functions.
- **Limited Libraries**: We only open essential Lua libraries (`base`, `math`, `table`). We deliberately **do not** open potentially dangerous libraries like `os` (which can execute system commands) or `io` (which can read/write files).

- **Alternatives Considered**:
- **Loading C++ DLLs as Plugins for Game Logic**: This is extremely powerful but also extremely dangerous from a security standpoint. A malicious DLL would have full access to the server's memory and could do anything. This is only viable in a trusted environment (e.g., for official DLC).
- **Custom Scripting Language**: Designing our own language would provide perfect security, as we would control every instruction.
- **Cons**: A monumental task, far beyond the project's scope, and would be less powerful and familiar than a mature language like Lua.

**Conclusion**: Using a properly sandboxed Lua environment provides the required flexibility for game design while containing potential security risks by exposing a minimal and controlled API.