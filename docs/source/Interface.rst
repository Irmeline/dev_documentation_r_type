==========================
Engine Interface Reference
==========================

This document provides a detailed reference for all the abstract interfaces that a backend plugin must implement to be compatible with the engine.

A complete backend must provide a concrete implementation for every class and structure listed here.

---

Core Interfaces
---------------

``IGraphicFactory``
   The central factory responsible for creating all other concrete graphics and audio objects. It is the main entry point for a backend plugin.

   .. code-block:: cpp
      :caption: Engine/include/Graphics/IGraphicFactory.hpp

      class IGraphicFactory {
      public:
          virtual ~IGraphicFactory() = default;

          virtual std::unique_ptr<IRenderWindow> createWindow(...) = 0;
          virtual std::unique_ptr<ITexture> createTexture() = 0;
          virtual std::unique_ptr<ISprite> createSprite() = 0;
          virtual std::unique_ptr<IFont> createFont() = 0;
          virtual std::unique_ptr<IText> createText() = 0;
          virtual std::unique_ptr<ISoundBuffer> createSoundBuffer() = 0;
          virtual std::unique_ptr<ISound> createSound() = 0;
      };

``Event`` (struct)
   A generic structure used to represent window and input events in a library-agnostic way. It uses an ``EventType`` enum and a union to hold event-specific data.

   .. code-block:: cpp
      :caption: Engine/include/Graphics/IEvent.hpp

      enum class EventType {
          Closed,
          Resized,
          KeyPressed,
          KeyReleased,
          MouseButtonPressed,
          MouseButtonReleased,
          MouseMoved,
          Unknown
      };

      enum class KeyCode {
          A, B, C, D, E, F, G, H, I, J, K, L, M, N, O, P, Q, R, S, T, U, V, W, X, Y, Z,
          Num0, Num1, Num2, Num3, Num4, Num5, Num6, Num7, Num8, Num9,
          Escape, Enter, Space, Backspace, Tab,
          Left, Right, Up, Down,
          Unknown
      };

      enum class MouseButton {
          Left, Right, Middle,
          Unknown
      };

      struct Event {
          EventType type = EventType::Unknown;
          union {
              struct {
                  KeyCode code;
                  bool alt;
                  bool control;
                  bool shift;
              } key;

              struct {
                  MouseButton button;
                  int x;
                  int y;
              } mouseButton;
              
              struct {
                  int x;
                  int y;
              } mouseMove;

              struct {
                  unsigned int width;
                  unsigned int height;
              } size;
          };
      };

---

Graphics Interfaces
-------------------

``IRenderWindow``
   Represents the main application window. It handles its lifecycle, event polling, view management, and provides access to the renderer.
   
   .. code-block:: cpp
      :caption: Engine/include/Graphics/IRenderWindow.hpp

      class IRenderWindow {
      public:
          virtual ~IRenderWindow() = default;

          virtual bool isOpen() const = 0;
          virtual void close() = 0;
          virtual bool pollEvent(Event& event) = 0;
          virtual void clear(uint8_t r = 0, uint8_t g = 0, uint8_t b = 0) = 0;
          virtual void display() = 0;
          virtual IRenderer& getRenderer() = 0;
          virtual void setView(float centerX, float centerY, float width, float height) = 0;
          virtual bool hasFocus() const = 0;
          virtual void setFramerateLimit(unsigned int limit) = 0;
          virtual bool isKeyPressed(KeyCode key) const = 0;
          virtual bool isMouseButtonPressed(MouseButton button) const = 0;
          virtual std::pair<int, int> getMousePosition() const = 0;
      };

``IRenderer``
   Represents the "drawer". It exposes methods to render drawable objects like sprites and text onto its render target (the window).

   .. code-block:: cpp
      :caption: Engine/include/Graphics/IRenderer.hpp

      class IRenderer {
      public:
          virtual ~IRenderer() = default;
          virtual void draw(ISprite& sprite) = 0;
          virtual void draw(IText& text) = 0;
      };

``ITexture``
   Represents the raw pixel data of an image loaded in memory. Its primary role is to be loaded from a file and assigned to an `ISprite`.

   .. code-block:: cpp
      :caption: Engine/include/Graphics/ITexture.hpp

      class ITexture {
      public:
          virtual ~ITexture() = default;

          virtual bool loadFromFile(const std::string& filepath) = 0;
          virtual unsigned int getWidth() const = 0;
          virtual unsigned int getHeight() const = 0;
      };

``ISprite``
   Represents a drawable, textured object. It can be positioned, scaled, rotated, and its texture rectangle can be changed for animations.

   .. code-block:: cpp
      :caption: Engine/include/Graphics/Isprite.hpp

      struct IntRect {
          int left, top, width, height;
      };

      struct floatRect {
          float left, top, width, height;
      };

      class ISprite {
      public:
          virtual ~ISprite() = default;

          virtual void setTexture(ITexture& texture) = 0;
          virtual void setTextureRect(const IntRect& rect) = 0;
          virtual void setPosition(float x, float y) = 0;
          virtual void setScale(float x, float y) = 0;
          virtual void setRotation(float angle) = 0;
          virtual void setColor(uint8_t r, uint8_t g, uint8_t b, uint8_t a = 255) = 0;
          virtual floatRect getGlobalBounds() const = 0;
      };

``IFont``
   Represents the data of a font file (.ttf, etc.) loaded in memory.

   .. code-block:: cpp
      :caption: Engine/include/Graphics/IFont.hpp

      class IFont {
      public:
          virtual ~IFont() = default;
          virtual bool loadFromFile(const std::string& filepath) = 0;
      };

``IText``
   Represents a drawable text object. It requires an `IFont` to be rendered and can have its string content, size, and color modified.

   .. code-block:: cpp
      :caption: Engine/include/Graphics/IText.hpp

      class IText {
      public:
          virtual ~IText() = default;

          virtual void setFont(IFont& font) = 0;
          virtual void setString(const std::string& text) = 0;
          virtual void setCharacterSize(unsigned int size) = 0;
          virtual void setPosition(float x, float y) = 0;
          virtual void setFillColor(uint8_t r, uint8_t g, uint8_t b) = 0;
      };


---

Audio Interfaces
----------------

``ISoundBuffer``
   Represents the raw data of a short audio clip (an effect) loaded completely into memory.

   .. code-block:: cpp
      :caption: Engine/include/Audio/ISoundBuffer.hpp

      class ISoundBuffer {
      public:
          virtual ~ISoundBuffer() = default;
          virtual bool loadFromFile(const std::string& filepath) = 0;
      };


``ISound``
   Represents a "voice" or a "channel" that can play an `ISoundBuffer`.

   .. code-block:: cpp
      :caption: Engine/include/Audio/ISound.hpp

      enum class SoundStatus {
          Stopped,
          Paused,
          Playing
      };

      class ISound {
      public:
          virtual ~ISound() = default;

          virtual void setBuffer(ISoundBuffer& buffer) = 0;
          virtual void play() = 0;
          virtual void pause() = 0;
          virtual void stop() = 0;
          virtual void setVolume(float volume) = 0;
          virtual void setLoop(bool loop) = 0;
          virtual SoundStatus getStatus() const = 0;
      };

---

Implementing All Engine Interfaces
----------------------------------

A complete backend must provide a concrete implementation for **every interface** defined in the engine.