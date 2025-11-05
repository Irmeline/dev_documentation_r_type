.. R-TYPE Developer Documentation master file, created by
   sphinx-quickstart on Sat Oct 11 13:24:07 2025.

==================================
R-TYPE Developer Documentation
==================================

*R-TYPE* is a multiplayer shoot’em up video game developed in *C++20*, inspired by the classic arcade game *R-Type*.
The player controls a spaceship that must defeat waves of enemies and dodge incoming projectiles.

The main goal of this project is to design a **modular, reusable, and networked game engine** using an *ECS (Entity Component System)* architecture.
A core principle of this engine is its **complete independence from any specific graphics or audio library**.

- The **Engine** provides a generic framework for creating games.
- The **Client** is a thin application that loads a **graphics backend plugin** (e.g., for SFML or SDL) to handle rendering and input.
- The **Server** is a headless application that manages game logic, synchronization, and communication via *ASIO*.

This documentation is intended for developers joining the project.
It focuses on:
- Understanding the *high-level architecture* (Engine, Backends, Game Logic).
- Knowing how to *build and run* the project.
- Following *team conventions* and *code contribution rules*.

.. toctree::
   :maxdepth: 2
   :caption: Table of Contents:

   overview
   Architecture
   Network
   Lobby
   Logic_server
   client
   graphic
   Interface
   Comparative
   Contribution


---
