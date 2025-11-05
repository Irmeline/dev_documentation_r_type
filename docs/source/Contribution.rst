Contribution & Coding Guidelines
================================

**Code Style:**
- Follow **Epitech C++ standard** (header guards, naming, formatting).  
- Each system and component has its own `.hpp` and `.cpp`.

**Commit Rules:**
- Clear, imperative messages:  
Example: `feat(network): add packet serialization`

**Branch Naming:**
- `feature/<feature-name>` for new features  
- `fix/<issue-name>` for bug fixes

**Documentation:**
- Update **Sphinx docs** when adding systems or architecture changes.

**Example Workflow:**
1. `git checkout -b feature/movement-system`
2. Implement + test locally
3. Update docs and run `make html`
4. Open Pull Request

---

Build Instructions
==================

0. **checkups**
    - g++ maximum version: 15
    - Clang maximun version: 15

1. **Configure and build:**

   .. code-block:: bash

      mkdir build && cd build
      cmake ..
      make

2. **Run the server:**

   .. code-block:: bash

      ./r-type_server port

3. **Run the client:**

   .. code-block:: bash

      ./r-type_client addresse_IP port mode
      mode: |infinite|create|join

---

Credits
=======

**Developers:**  
- Océane **KODJELA** — ECS Engine & Systems  
- Zoltan **BABKO** — Networking & Server  
- Aurel **PLIYA**, Paul **MOURENS**, Vlad **BURGA** — Client & Graphics  

Developed as part of **Epitech G-CPP-500 (2025)**.

