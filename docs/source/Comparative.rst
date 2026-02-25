=============================
Comparative & Technical Study
=============================

This document presents the technical choices made during the development of the R-Type engine.
For each decision, we first present the available options with their respective strengths and weaknesses, then explain our conclusion.

---

1. Core Language
================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **C++20**
     - Industry standard for game engines. Direct hardware access. Templates, smart pointers, constexpr. Massive ecosystem. High performance.
     - Complex syntax. Manual memory management risks. Long compile times.
     - Excellent
   * - **Rust**
     - Memory safety guaranteed at compile time. No data races. Modern tooling (cargo).
     - Steeper learning curve. Less mature game dev ecosystem. Interop with C libraries less straightforward.
     - Good
   * - **C# (.NET / Unity)**
     - Rapid development. Garbage collection. Rich game frameworks (Unity, Godot).
     - Requires a runtime (GC pauses). Using Unity defeats the purpose of building an engine. Lower raw performance.
     - Poor

**Conclusion:** C++20 was selected because the project's goal is to build a game engine from scratch. C++ provides the necessary control over memory layout, which is essential for ECS performance (data-oriented design with cache-friendly ``sparse_array`` containers). Its ecosystem for networking (ASIO), scripting (sol2/Lua), and multimedia (SFML) is unmatched.

---

2. Architectural Pattern
========================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **ECS (Entity Component System)**
     - Data locality (cache-friendly). Composition over inheritance. Entities defined by data, not code. Easy to add new entity types via JSON.
     - Higher initial complexity. Requires discipline to separate data from logic.
     - Excellent
   * - **Traditional OOP (Inheritance)**
     - Familiar to most developers. Simple for small projects. Clear class hierarchies.
     - Diamond problem. Rigid hierarchies. Poor cache performance when iterating many objects. Adding new entity combinations requires new classes.
     - Poor
   * - **Component-Based (without Systems)**
     - Composition benefits. Components attached to game objects.
     - Logic often ends up inside components (coupling data and logic). No clear iteration model for bulk processing.
     - Moderate

**Conclusion:** ECS was selected for its superior performance characteristics and flexibility. In R-Type, hundreds of entities (bullets, enemies, particles) must be processed every frame. ECS allows systems to iterate over contiguous arrays of components, maximizing CPU cache utilization. New entity types (enemies, powerups) can be created entirely through JSON configuration without writing new C++ code.

---

3. Graphics & Audio Abstraction
===============================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **Abstract Interfaces + Dynamic Plugin Loading**
     - Engine is 100% independent of any library. Backend can be swapped at runtime. Clean separation of concerns.
     - Most complex to implement. Slight virtual call overhead. Requires a plugin entry point.
     - Excellent
   * - **Direct Library Integration (e.g., SFML everywhere)**
     - Fastest to implement. No abstraction overhead. Direct access to all library features.
     - Engine permanently coupled to one library. Porting requires rewriting engine code. Fails the "reusable engine" requirement.
     - Poor
   * - **Compile-Time Abstraction (#ifdef)**
     - No runtime overhead. Can support multiple libraries.
     - Messy preprocessor code. Must recompile entire project to switch. Cannot support two backends simultaneously.
     - Moderate

**Conclusion:** The plugin-based approach was selected because the project specification requires a **reusable engine independent of any specific library**. We defined abstract interfaces (``IGraphicsFactory``, ``IRenderWindow``, ``ISprite``, ``IText``, ``ISound``, etc.) and implemented them in a SFML backend compiled as a shared library (``.so``). The ``PluginLoader`` loads this library at runtime. Switching to SDL or Raylib only requires implementing the interfaces in a new shared library. The engine code remains untouched.

---

4. Networking Protocol
======================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **Custom Binary UDP + Compression + ACK**
     - Minimal latency. Compact packets. Full control over packet structure. Delta-based snapshots save bandwidth. zlib compression reduces packet size.
     - Must implement reliability manually. No built-in ordering guarantees.
     - Excellent
   * - **TCP**
     - Reliable, ordered delivery. Simple to use.
     - Head-of-line blocking causes lag spikes. Higher latency. Unsuitable for real-time action games.
     - Poor
   * - **Reliable UDP Libraries (ENet, RakNet)**
     - Built-in reliable channels. Proven in production games. Less code to write.
     - Additional dependency. Less control over protocol details. May include unnecessary features.
     - Good
   * - **Text-Based Protocol (JSON over UDP)**
     - Human-readable. Easy to debug.
     - Much larger packet sizes. Slower to parse. Wastes bandwidth on field names and formatting.
     - Poor

**Conclusion:** We chose a custom binary UDP protocol with zlib compression and a lightweight ACK/retransmission system. Each game tick, the server builds a **delta snapshot** containing only the changes (creations, destructions, position modifications, player states). This snapshot is serialized into a compact binary format, compressed with zlib, and sent to all clients. Clients send back ACK packets for reliable delivery of critical data. For a game where a single frame's position data quickly becomes obsolete, UDP's low latency is far more valuable than TCP's guaranteed delivery.

**Protocol Stack:**

.. code-block:: text

   Application:  [opcode][sequence][flags][destructions][creations][modifications][players]
   Compression:  [opcode_byte] + [zlib_compressed_payload]
   Transport:    UDP datagram
   Reliability:  ACK packets + retransmission after 1 second timeout

---

5. Networking Library
=====================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **ASIO (Standalone)**
     - High performance. Asynchronous, non-blocking I/O. Cross-platform. Header-only. No dependency on Boost.
     - Steeper learning curve (callback-based). Verbose API.
     - Excellent
   * - **Boost.Asio**
     - Same as standalone ASIO. Part of the Boost ecosystem.
     - Pulls in the entire Boost dependency. Larger build times. Overkill for this project.
     - Good
   * - **Raw BSD Sockets**
     - Maximum control. No dependencies.
     - Platform-specific code needed. No async support without manual threading. Error-prone.
     - Moderate

**Conclusion:** Standalone ASIO was selected for its non-blocking asynchronous model, which allows the server to handle network I/O in a dedicated thread without ever blocking the game loop. The ``async_receive_from`` and ``async_send_to`` calls integrate perfectly with our per-room ``GameServer`` architecture where each room's network runs on the shared ``io_context``.

---

6. Scripting Language
=====================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **Lua (via sol2)**
     - Extremely lightweight and fast. Designed for embedding. Industry standard for game scripting. sol2 makes C++ binding trivial and type-safe.
     - Limited standard library. Less familiar to some developers.
     - Excellent
   * - **No Scripting (Hardcoded C++)**
     - No additional language to learn. Maximum performance.
     - Every level change requires recompilation. Extremely inflexible for iterative game design.
     - Poor
   * - **Python (via pybind11)**
     - Large standard library. Familiar to many developers.
     - Heavier runtime. Slower execution. Larger memory footprint. Not designed for real-time embedding.
     - Moderate
   * - **JSON-based Logic**
     - Simple format. Easy to parse.
     - Not a programming language. No conditionals, loops, or functions. Becomes unmanageable for complex level logic.
     - Poor

**Conclusion:** Lua with sol2 was selected because level design and enemy wave patterns need frequent iteration. Our ``GameRulesSystem`` creates a sandboxed Lua environment and exposes C++ functions (``create_entity``, ``spawn_powerup``, ``get_random_position``, ``notify_level_completed``, etc.). Level designers write ``.lua`` scripts that define an ``update(deltaTime)`` function called every tick. This enables creating complex level scenarios, from timed enemy waves to infinite mode with scaling difficulty, without touching C++ code.

---

7. Data Compression
===================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **zlib (deflate)**
     - Industry standard. Fast compression/decompression. Good compression ratios. Widely available.
     - Requires linking against zlib library.
     - Excellent
   * - **LZ4**
     - Extremely fast decompression. Low CPU overhead.
     - Lower compression ratio than zlib. Additional dependency.
     - Good
   * - **No Compression**
     - Zero CPU overhead. Simplest implementation.
     - Larger packets. Higher bandwidth usage. May exceed UDP MTU for large snapshots.
     - Moderate

**Conclusion:** zlib was selected because game snapshots with many entity modifications can exceed the typical UDP MTU (1500 bytes). zlib's deflate algorithm provides good compression ratios on our structured binary data, reducing packet sizes significantly. The ``Datacompression`` utility wraps zlib's ``compress2`` and ``uncompress`` functions. The compressed payload is prefixed with the original size for decompression.

---

8. Build System & Dependencies
==============================

.. list-table::
   :widths: 15 30 30 25
   :header-rows: 1

   * - Option
     - Strengths
     - Weaknesses
     - Fit for Project
   * - **CMake + Conan**
     - CMake is the C++ standard. Conan handles cross-platform dependency management. Reproducible builds.
     - CMake syntax can be verbose. Conan adds a setup step.
     - Excellent
   * - **Makefile**
     - Simple for small projects. No extra tools.
     - Hard to maintain for large projects. No cross-platform support. Manual dependency management.
     - Poor
   * - **Meson + vcpkg**
     - Meson has cleaner syntax. vcpkg integrates well with Visual Studio.
     - Less widespread than CMake in C++ ecosystem. vcpkg is more Windows-focused.
     - Good

**Conclusion:** CMake was selected as the industry standard for C++ projects. Conan manages external dependencies (SFML, ASIO, sol2, nlohmann_json, zlib) ensuring consistent builds across Linux, macOS, and Windows.

---

Summary Table
=============

.. list-table::
   :widths: 25 25 50
   :header-rows: 1

   * - Domain
     - Selected
     - Key Reason
   * - Language
     - C++20
     - Performance, control, ecosystem
   * - Architecture
     - ECS
     - Cache-friendly, data-driven, flexible
   * - Graphics
     - Abstract Interfaces + Plugins
     - Engine independence, runtime backend swap
   * - Network Protocol
     - Custom Binary UDP
     - Low latency, compact, delta snapshots
   * - Network Library
     - ASIO (Standalone)
     - Async non-blocking I/O, cross-platform
   * - Scripting
     - Lua + sol2
     - Lightweight, fast, ideal for level design
   * - Compression
     - zlib
     - Standard, good ratio, widely available
   * - Build System
     - CMake + Conan
     - Industry standard, cross-platform deps