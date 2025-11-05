====================================
Extending the Engine: New Backends
====================================

The engine's core strength is its graphics and audio abstraction layer. This allows any developer to add support for a new multimedia library (like SDL, Raylib, or Allegro) without modifying the engine's source code.

This guide explains the process of creating a new **backend plugin**. We will use **SDL2** as an example.

The Process Overview
--------------------
0.  **Manage Dependencies**: Ensure the new library (e.g., SDL2) is available by adding it to the project's ``conanfile.txt`` and configuring CMake.
1.  **Create a New Project**: Set up a new shared library project for the backend (e.g., ``SDLBackend``).
2.  **Implement Interfaces**: Create concrete classes (e.g., `SDLWindow`, `SDLSprite`) that inherit from the engine's abstract interfaces (`IRenderWindow`, `ISprite`).
3.  **Implement the Factory**: Create an `SDLFactory` that knows how to construct all your concrete SDL classes.
4.  **Create the Entry Point**: Expose a C-style `createFactory()` function that the `PluginLoader` can find.
5.  **Compile and Use**: Compile the project into a ``.so`` or ``.dll`` and update the client's `main.cpp` to load the new plugin.

---

Step 0: Managing Dependencies (Conan & CMake)
==============================================

If the new library (like SDL2) is not expected to be installed on the user's system, you must add it to the project's dependency manager.

**1. Update `conanfile.txt`**
  Add the new library to the ``[requires]`` section of your root ``conanfile.txt``. Conan will handle downloading and compiling it.

  .. code-block:: text
     :caption: conanfile.txt

     [requires]
     sfml/2.6.1
     asio/1.28.1
     sol2/3.3.1
     nlohmann_json/3.11.3
     sdl/2.30.0  # <--- ADD THE NEW LIBRARY HERE

**2. Update `CMakeLists.txt`**
  In the main `CMakeLists.txt`, you need to tell CMake to find the new package provided by Conan. Then, in the `CMakeLists.txt` for your new backend, you must link against it.

  .. code-block:: cmake
     :caption: Root CMakeLists.txt

     # ... (after other find_package calls)
     find_package(SDL2 REQUIRED)

  .. code-block:: cmake
     :caption: Backends/SDLBackend/CMakeLists.txt

     # ...
     # Link against the SDL2 library found by Conan
     target_link_libraries(SDLBackend PRIVATE
         EngineCore
         SDL2::SDL2
     )

---

Step 1: Implementing an Interface (Example: `ISprite`)
========================================================

Each interface from the engine must have a corresponding concrete implementation in the new backend. Let's take `ISprite` as an example.

The goal is to "wrap" the native SDL object (`SDL_Texture*` and `SDL_Rect`) inside a class that conforms to the `ISprite` interface.

.. code-block:: cpp
   :caption: Backends/SDLBackend/include/SDLSprite.hpp

   #pragma once
   #include "Engine/Graphics/ISprite.hpp"
   #include <SDL2/SDL.h> // Include the specific library's headers

   class SDLTexture; // Forward-declare the concrete texture class

   class SDLSprite : public ISprite {
   public:
       SDLSprite(SDL_Renderer* renderer); // Needs the SDL renderer to draw itself

       // --- Interface Implementation ---
       void setTexture(ITexture& texture) override;
       void setTextureRect(const IntRect& rect) override;
       void setPosition(float x, float y) override;
       // ... (implement all other ISprite methods)

       // --- Helper for the SDL Renderer ---
       void render();

   private:
       SDL_Renderer* _renderer;
       SDL_Texture* _texture = nullptr; // Pointer to the native texture
       SDL_Rect _sourceRect;            // The portion of the texture to draw
       SDL_FRect _destRect;              // Where on the screen to draw (with size)
   };

The ``.cpp`` file would then contain the "translation" logic.

.. code-block:: cpp
   :caption: Backends/SDLBackend/src/SDLSprite.cpp

   #include "SDLSprite.hpp"
   #include "SDLTexture.hpp" // Needed for the static_cast

   void SDLSprite::setTexture(ITexture& texture) {
       // We downcast the abstract ITexture to our concrete SDLTexture
       // to get access to the native SDL_Texture object.
       _texture = static_cast<SDLTexture&>(texture)._texture;

       // We query the texture to get its original size
       SDL_QueryTexture(_texture, NULL, NULL, &_sourceRect.w, &_sourceRect.h);
       _destRect.w = _sourceRect.w;
       _destRect.h = _sourceRect.h;
   }
   
   void SDLSprite::setPosition(float x, float y) {
       _destRect.x = x;
       _destRect.y = y;
   }

   void SDLSprite::render() {
       if (_texture) {
           SDL_RenderCopyF(_renderer, _texture, &_sourceRect, &_destRect);
       }
   }

This process is repeated for **all** interfaces: `ITexture`, `IRenderWindow`, `IFont`, etc.

---

Step 2: Implementing the Factory (`IGraphicFactory`)
====================================================

The factory is the central piece that knows how to create all the concrete objects for your backend.

.. code-block:: cpp
   :caption: Backends/SDLBackend/include/SDLFactory.hpp

      #pragma once
      #include "Engine/Graphics/IGraphicFactory.hpp"

       class SDLFactory : public IGraphicFactory {
       public:
            // The factory needs to know about the global SDL renderer
            SDLFactory(SDL_Renderer* renderer);

            // --- Interface Implementation ---
            std::unique_ptr<ISprite> createSprite() override;
            std::unique_ptr<ITexture> createTexture() override;
            // ... (implement creation methods for all interfaces)
        
        private:
            SDL_Renderer* _renderer;
        };


.. code-block:: cpp
   :caption: Backends/SDLBackend/src/SDLFactory.cpp

   #include "SDLFactory.hpp"
   #include "SDLSprite.hpp"
   #include "SDLTexture.hpp"

   std::unique_ptr<ISprite> SDLFactory::createSprite() {
       // When the engine asks for an ISprite, we return our concrete SDLSprite.
       return std::make_unique<SDLSprite>(_renderer);
   }
   
   std::unique_ptr<ITexture> SDLFactory::createTexture() {
       return std::make_unique<SDLTexture>(_renderer);
   }

   // ...

Step 3: Creating the Plugin Entry Point
=======================================

This is the most critical step. You must expose a simple, C-style function with a known name (createFactory) that the engine's PluginLoader can find inside the compiled .so or .dll file.

.. code-block:: cpp
   :caption: Backends/SDLBackend/src/SDLFactory.cpp (at the end)

   // The entry point function
   extern "C" DLLEXPORT IGraphicFactory* createFactory() {
       // This function is responsible for the initial setup of the library
       if (SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO) < 0) {
           // Handle error
           return nullptr;
       }

       // ... (Create window and renderer)

       // Create and return an instance of our concrete factory.
       return new SDLFactory(g_renderer);
   }

---

Step 4: Compiling and Using the New Backend
===========================================

1.  **CMake**: Create a `CMakeLists.txt` for the `SDLBackend` project. It should be configured to build a **SHARED** library and link against the SDL2 libraries.

2.  **`main.cpp`**: In the client's `main.cpp`, change the path passed to the `PluginLoader`.

    .. code-block:: cpp
       :caption: main.cpp (Client)

       // OLD:
       // PluginLoader plugin("./lib/libSFMLBackend.so");
       
       // NEW:
       PluginLoader plugin("./lib/libSDLBackend.so");

       // The rest of the code remains UNCHANGED.
       std::unique_ptr<IGraphicFactory> factory = plugin.createFactory();
       auto window = factory->createWindow(1920, 1080, "R-Type Client");
       // ...

By following this pattern, any developer can add support for a new library, proving the engine's modularity and extensibility.