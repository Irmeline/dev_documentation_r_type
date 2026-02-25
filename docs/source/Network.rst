Networking
==========

The project uses a client-server architecture with a **server-authoritative model**. Communication is handled via **UDP** using the **ASIO** library. All game data is compressed with **zlib** and uses a lightweight **ACK/retransmission** mechanism for reliability.

Connection Flow
---------------

1. The **Client** connects to the **Lobby** on the main port (default: 3000).
2. The client sends a ``HELLO`` packet, then chooses a mode: ``CREATE_ROOM``, ``JOIN_ROOM``, or ``CREATE_INFINITE``.
3. The **Lobby** creates or finds a ``GameServer`` instance on a dedicated port.
4. The Lobby sends a ``JOIN_SUCCESS`` packet containing the game port.
5. The **Client** connects to the game port and sends a ``CLIENT_INPUT`` packet (which also serves as the initial handshake).
6. The **Server** responds with a ``WELCOME`` packet containing the player's entity ID.
7. The game loop begins: the server broadcasts **compressed delta snapshots** at a fixed tick rate.

Protocol
--------

All in-game packets use a custom binary protocol over UDP. The server sends a single **snapshot packet** per tick containing only the changes that occurred during that tick.

**Snapshot Packet Structure:**

.. code-block:: text

   ┌─────────┬──────────┬───────┬────────┬────────┬────────┬─────────┐
   │ opcode  │ sequence │ flags │ dest_n │ crea_n │ mod_n  │ play_n  │
   │ (u8)    │ (u32)    │ (u8)  │ (u16)  │ (u16)  │ (u16)  │ (u16)   │
   └─────────┴──────────┴───────┴────────┴────────┴────────┴─────────┘
   ├─ destructions: [entityId (u32)] × dest_n
   ├─ creations:    [entityId (u32), typeId (u8), x (f32), y (f32)] × crea_n
   ├─ modifications:[entityId (u32), x (f32), y (f32)] × mod_n
   └─ player states:[entityId (u32), hp (u32), score (u32)] × play_n

**Game State Flags (bitfield):**

.. list-table::
   :widths: 20 10 40
   :header-rows: 1

   * - Flag
     - Value
     - Description
   * - ``NONE``
     - 0x00
     - Normal gameplay
   * - ``LEVEL_COMPLETED``
     - 0x01
     - Current level beaten, transition to next
   * - ``GAME_VICTORY``
     - 0x02
     - All levels completed
   * - ``GAME_OVER``
     - 0x04
     - All players dead

Compression
-----------

Before transmission, the snapshot binary payload is compressed using **zlib** (deflate algorithm). The final UDP packet has the following structure:

.. code-block:: text

   [opcode_byte (1 byte)] + [zlib_compressed_payload (N bytes)]

The client reads the first byte to identify the packet type, then decompresses the remaining bytes before parsing the snapshot.

Reliable Delivery (ACK System)
------------------------------

The server assigns a **sequence number** to each snapshot. When a client receives a snapshot, it sends back an ``ACK`` packet containing the sequence number. If the server does not receive an ACK within 1 second, it retransmits the packet. This ensures that critical state changes (entity creation, destruction) are eventually delivered.

.. code-block:: text

   Server → Client:  GAME_SNAPSHOT (seq=42, compressed)
   Client → Server:  ACK (ackSequence=42)

   If no ACK received after 1000ms:
   Server → Client:  GAME_SNAPSHOT (seq=42, retransmit)

Pre-Filtering (Packet Integrity)
---------------------------------

A critical rule in the snapshot builder is that **the header count must exactly match the number of entries written**. Before building the packet, the server pre-filters all data:

- **Destructions**: Deduplicated (no duplicate entity IDs).
- **Same-frame entities**: If an entity is both created and destroyed in the same tick, it is removed from both lists and silently killed server-side. This prevents "ghost entities" on the client.
- **Creations**: Entities with ``UNKNOWN`` type are excluded.
- **Modifications**: Entities without a ``Position`` component, destroyed entities, and same-frame entities are excluded.
- **Player states**: Only entities with both ``Player`` and ``Health`` components are included.

This filtering happens **before** the header is written, ensuring zero skips in the serialization loop.

Client Input
------------

Client inputs are sent as small, frequent UDP packets containing a bitmask of pressed keys:

.. code-block:: text

   ┌─────────┬────────────┐
   │ opcode  │ input_mask │
   │ (u8)    │ (u8)       │
   └─────────┴────────────┘

   Bit 0: UP
   Bit 1: DOWN
   Bit 2: LEFT
   Bit 3: RIGHT
   Bit 4: SHOOT

Key Classes
-----------

``ServerNetworkSystem``
  Engine-side system that broadcasts compressed delta snapshots to all connected clients. Handles client connections, welcome packets, ACK tracking, and retransmission.

``ClientNetworkSystem``
  Client-side system that receives compressed snapshots, decompresses them, parses the header and payload, and applies destructions/creations/modifications to the local ``Registry``. Also handles kick/ban packets from the lobby.

``Datacompression``
  Utility class wrapping zlib's ``compress2`` and ``uncompress`` functions for snapshot compression.

``Serialization``
  Namespace providing binary read/write helpers: ``write_u8``, ``write_u16``, ``write_u32``, ``write_f32``, and their ``read_`` counterparts.