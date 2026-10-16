---
layout: page
title: How Minecraft runs
permalink: /minecraft/
---

# /How Minecraft runs

This page will cover the very basics of how Minecraft runs. It's a bit of an abstract concept from a player point of view, so please bear with me. If you look at the game, but strip away all of the actual content, nothing's left, except for some of the glue that held it all together and the pedestal it stood on. That's what we'll talk about.

Every Java program needs some entry point to start from, in the case of Minecraft that's the aptly named `Minecraft` class found in `net.minecraft.client` (note: not quite, the launcher's still involved here, but that's not something you are ever going to interact with). This 3000 line behemoth does a lot of things, but we'll only focus on the basics for now. (Another note: this works differently on dedicated servers, where most of the basics are handled by a class named `MinecraftServer`, more on that later)

## Startup
Does about what one would expect it to do, when Minecraft starts, it sets up all the most basic things:

* Runs `Bootstrap.func_151354_b()`, a function that triggers all blocks and items to be registered, along with item and block statistics
* Loads the game settings from the options file
* Sets up the window in which the game is rendered
* Loads the default resource pack into memory
* Does a whole lot of OpenGL rendering setup which is too complicated to explain
* Creates instances of many singleton manager-type classes, like utils for loading textures, handling the wodl save format, the font renderer, stuff like that
* Initializes the sound engine
* Loads the texture maps (also known as "texture atlas"), which are all the item or block textures stitched together into single images
* Loads up the main menu GUI

The thread in which this happens is the client thread. As the sound engine starts, it will create an audio thread. As you start a world or join a server, a few additional threads will pop up, that being the server thread, a few network related threads, and a few others that are less important for now.

The client thread contains the client portion of the game, i.e. handling key input, doing all the rendering, and running the client portion of the world. The server thread only runs the server portion of the world. The client and server worlds do have a lot of overlap for reasons we'll get into later, for now it's enough to understand that certain things in the world only need to happen on the client, some only on the server, and some are done by both in order to stay in sync without having to constantly send information between the two.

The sound engine nad network threads are not something you typically have to work with directly, playing sounds and sending network packets is done from one of the previously mentioned world threads.

## Game loop
As soon as the client game has started, it enters the main game loop. Each cycle of this loop is one frame. These are the actual frames you experience during gameplay, and which are affected by vsync or FPS limits, so reducing FPS artificially will also affect anything within the client's game loop, like key inputs.

The main game loop will:

* Update the game's internal timer which is important for rendering
* Do `runTick()`, which is the client's world tick component which updates the world time, entities, tile entities, player movement, basically almost everything ingame that moves. It will likely not do this every cycle, as the games simulation speed (TPS) is capped at 20, and the client's framerate is usually higher
* Bind the frame buffer (that's more rendering stuff)
* Call `entityRenderer.updateCameraAndRender` which does almost all of the ingame rendering (or rather, all of the stuff we actually care about)
* Do various things related to the F6 Twitch stream integration
* Update the profiler (i.e. the thing that gives insight on performance and what parts eat up the most frame time), although no one knows why it does this because Minecraft's built-in profiler is garbage
* If the game loop has concluded in less time than the frame limit allows, it will wait until the minimum time has elapsed (on a frame limit of 100 FPS, it will wait until the full loop cycle has taken at least 0.01s)

## Tick loop
This is the part that is most relevant to what's happening ingame, as it handles all the updates for the things we can actually see and interact with. The tick loop is fixed to run at 20 cycles per second, although it can run more often if the game has previously frozen for a bit, causing scheduled ticks to "build up" and then catch up again.

The tick loop:

* Handles all the user input, whether it being the mouse moving, clicking, various keybinds
* Updates certain per-tick render components like the surrounding brightness, torch flicker, the current shown boss bar and ticking the item equip/unequip progress
* Runs `updateEntities()` on the world instance, which ticks all weather effects (lightning), entities (like NPCs or projectiles) and tile entities (blocks with extra data like furnaces and beacons)
* Updates the music ticker which schedules random music tracks and the sound handler which can start scheduled sounds and runs the sound update routine on things like looped sounds
* Increments the world timer

## So what?
All of this is a lot to take in for a start, and may not even be all that useful right away. However, it can be helpful to at least have some general idea of how the game ticks work and in what order things happen within a tick. Also do note that all of this only really includes the main loop for the client game, dedicated servers work a little different.

