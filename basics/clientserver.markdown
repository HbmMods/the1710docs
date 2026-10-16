---
layout: page
title: Client/Server Concept
permalink: /clientserver/
---

# /Client/Server Concept

Requires:

* [How Minecraft runs](/the1710docs/minecraft)

The client/server concept is one of the most important topics you have to know about. Almost every single thing you want to make involves it, and requires considerations to be made because of it.

As you know by now, Minecraft has a client thread and a server thread, with both threads handling things in the world, sometimes these things are unique to that thread, sometimes things are done by both threads. This has a simple practical reason: This way, there's not much distinction between a singleplayer world and a server you've joined from the point of view of the client, which makes things simpler.

Back in ye olden days, server and client play was fundamentally different, and when making a mod, often times the server version would have to be different from the one loaded by a client. In release version 1.3, Mojang changed singleplayer from a single world thread (which had funny side effects such as the game freezing entirely when the world thread is busy) to a client and integrated server thread (which has funny side effects such as a singleplayer game rarely desyncing). The main consequence is that we now have to treat the game like you're playing on a server, no matter if you actually do.

## Of Servers and Clients

What does the server do, and what the client?

The server...

* ...runs most of the logic. This is a very broad definition, but if it's something that concerns all players, the server likely handles it itself. The progress of a furnace, the damage dealt by a skeleton's arrow, the direction a minecart is moving in, all of these things are decided by the server and the results are sent to the individual players.
* ...handles gamerules and all the other per-world things, like deciding the current weather and doing chunk generation.

The client...

* ...does all the things that are obvious for the client, like rendering, audio, player input.
* ...runs some logic that the server also runs, usually things that are deterministic, i.e. always have the same result. Crafting recipes are already the same on both client on server, so it is not exactly necessary to wait for the server to send a packet with the output, the client and server will both run that logic, the server only additionally sends validation (this for example is what causes disappearing items after crafting them when the recipes do desync, usually from differing configs). Update loops of entities and tile entities are also run by the client, although usually the behavior is different for which side is running it, the server might do the logic while the client does certain visual effects.
* ...makes sense of some of the things the server does which wouldn't work otherwise. For example, it would be way too expensive to send full position/angle/movement data for every single entity all the time. Therefore, most entities are only synced in short intervals and only if values change. So instead of mobs constantly teleporting short distances when running towards you, the client will constantly approach the true server-determined position via interpolation (i.e. "smoothening it out"), making movement fluid. This is also why endermen often look like they zoom to a new position when teleporting small distances, they don't actually run at light speed, rather it's the client's position interpolation that's kicking in, trying to smoothen out the apparent "movement".

## Is remote?

The most immediate way you'll get exposed to this concept is `World.isRemote`. This is a boolean field which is part of the world instance in the current context. `isRemote` is `true` for the client world and `false` for the server world. For example, since the tile entity update function is part of both the client and server, mob spawners use this function if `worldObj.isRemote` to spawn fire and smoke particles, and if `!worldObj.isRemote` to actually spawn the mobs.

## Know your limits

Naturally, given the differing roles of the client and the server, it's important to know where to do what:

* Placing a block or spawning an entity only happens on the server. The server will automatically notify the client. In fact, doing these things on the client can create temporary ghost blocks and unmoving entities with no AI.
* Spawning a particle can only happen on the client. The server has a few events by default which will send a packet to nearby clients, causing the particles to be spawned. While the function exists on servers, it will simply not do anything.
* Playing a sound is a bit more tricky. While sounds are client matter, the world instance has a function that automatically sends a packet from the server to the client for playing sounds. This will no nothing when done on the client. However, it is still possible to play sounds on the client simply by creating a new sound instance with the desired name and passing it to Minecraft's sound handler.

## Proxies

It may not be good form to start talking about Forge things when we haven't talked about Forge in general, but this one's somewhat simple and important for our topic: Proxies. A proxy class is a class that we register during startup, and which differs depending on whether we're a client or a server. How to register and use a proxy class is not terribly important right now, we'll go over that later, what's important now is understanding the things it does.

Servers cannot do rendering, after all they don't have a screen or a player playing the server itself, there's only clients playing *on* the server. Therefore, when started on a server, Forge will omit things only the client can do, like rendering, resource loading and audio, entirely. As in, during runtime, these things cease to exist. This is done using the `@SideOnly(Side.CLIENT)`
 annotation. You can find this annotation over most rendering classes, particle entity classes, GUIs and similar things.
 
 Imagine your mod is currently in its startup phase, and you're trying to register the renderers, i.e. assigning the classes that render entities to the corresponding entity classes. A client will do this no problem, however, the server will crash immediately. It would have to instantiate a renderer class, that from its point of view does not exist, due to the `@SideOnly`
 annotation. How would one go around this?
 
 Forge's answer is the proxies. Typically, there's a common (or server) proxy class which contains mostly methods with empty method bodies (NOPs), and a client proxy, a child class of the common proxy, which implements the methods. One such method could be used to register rendering for example, which handles all the hazardous client-only classes which are absent in the common proxy. In the mod's main registry, there's usually a reference field to the proxy using the `@SidedProxy` annotation, which uses class loading to automatically pick the right proxy for the current context. You don't have to know about how this is done in detail or remember this annotation right now, for now it's enough if you understand roughly what a proxy is and what it's used for.
 
The proxy can also provide us with a small bit of convenience: Imagine we want to spawn a particle. If we're on a client already, we only have to instantiate a particle entity and add it to the game's effect renderer instance. If we're on a server, we have to send a packet to tell the client to do just that instead. Instead of writing two separate utilities for something that does the same thing, jsut in different contexts, we could implement these things in both client and common proxies, and then simply call the current proxy's particle function. Depending on the context, it will now handle particle spawning, without us having to add any extra checks, since the current proxy always depends on whether the context is a server or client anyway.

Vanilla Minecraft has a somewhat similar approach with its world class, while most of the base functionality is in the abstract `World` base class, the client and server have different concrete implementations, that being `WorldClient` and `WorldServer`. This for example is the reason why in an earlier example, spawning firework particles does not work on the server, the abstract `World` defines `makeFireworks()` with an empty method body, resulting in a NOP on `WorldServer`, but `WorldClient` overrides this method and actually creates the relevant fireworks particles.

