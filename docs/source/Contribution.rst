Contribution & Coding Guidelines
================================

**Code Style:**

- Follow **Epitech C++ standard** (header guards, naming, formatting).
- Each system and component has its own ``.hpp`` and ``.cpp``.
- Use modern C++20 features: smart pointers, structured bindings, ``constexpr``.
- No raw ``new``/``delete``. Use ``std::unique_ptr`` and ``std::make_unique``.

**Commit Rules:**

- Clear, imperative messages.
- Example: ``feat(network): add packet compression with zlib``

**Branch Naming:**

- ``feature/<feature-name>`` for new features
- ``fix/<issue-name>`` for bug fixes

**Documentation:**

- Update **Sphinx docs** when adding systems or architecture changes.

**Example Workflow:**

1. ``git checkout -b feature/movement-system``
2. Implement + test locally
3. Update docs and run ``make html``
4. Open Pull Request

---

Build Instructions
==================

**Prerequisites:**

- g++ (max version 15) or Clang (max version 15)
- CMake 3.16+
- Conan 2.x (for dependency management)
- SFML 2.6, ASIO, sol2, nlohmann_json, zlib (managed by Conan)

**1. Install dependencies:**

.. code-block:: bash

   conan install . --build=missing

**2. Configure and build:**

.. code-block:: bash

   mkdir build && cd build
   cmake ..
   make

This produces three targets:

- ``r-type_server`` — the server executable
- ``r-type_client`` — the client executable
- ``libSFMLBackend.so`` — the SFML graphics backend plugin

**3. Run the server:**

.. code-block:: bash

   ./r-type_server [port]

Default port is 3000. The admin console starts automatically.

**4. Run the client:**

.. code-block:: bash

   ./r-type_client

The client opens a graphical menu. From there, the player can:

- Enter the server IP and port
- Create a multiplayer room or join an existing one
- Start an infinite (solo) mode game

---

Project Layout
==============

.. code-block:: text

   R-Type/
   ├── engine/          # Core ECS engine (static library)
   ├── Server/          # Server executable + game logic
   ├── Client/          # Client executable + menu + assets
   ├── Backends/        # Graphics backend plugins (SFML)
   ├── docs/            # Sphinx documentation (.rst files)
   ├── CMakeLists.txt   # Root build configuration
   └── conanfile.txt    # Dependency declarations

---

Credits
=======

**Developers:**

- Océane **KODJELA** — ECS Engine & Systems
- Zoltan **BABKO** — Networking & Server
- Aurel **PLIYA**, Paul **MOURENS**, Vlad **BURGA** — Client & Graphics

Developed as part of **Epitech G-CPP-500 (2025)**.