---
layout: page
title: Entities
permalink: /entities/
---

# /Entities

Requires:
* [Client/Server Concept](/the1710docs/clientserver)
* [Named Binary Tags (NBT)](/the1710docs/nbt)

Entities are objects that exist within the world that are neither blocks nor tile entities. Despite the name, tile entities and entities don't have much in common. Entities can move freely without being bound to the block grid (i.e. in increments smaller than a full block). In general, things that move are usually entities.

Examples for entities include:

* The player
* Mobs
* Projectiles like arrows and thrown bottles
* Fish hooks
* Hanged pictures, while placed like blocks, are entities, hence why the can occupy the same space as for example torches and slabs
* Ender crystals have a hitbox which can be attacked and damaged, and is therefore implemented as an entity, even though it too never moves and looks more like an animated block
* Dropped items, which intuitively would be items, are entities that just use an item stack for their appearance and to know what item the player gets upon being picked up
* XP orbs

Particles (`EntityFX`) technically extend the `Entity` class, but aren't really considered entities. Particles only exist on the client. Attempting to spawn particles on the server like they were regular entities will crash the game.

The base entity class has a lot of functionality that on its own, does absolutely nothing. Still, these things build the foundation of an entity and are used in many concrete implementations extensively:

* Height and width values, as well as an appropriately sized `AxisAlignedBB` instance that is this entity's hitbox
* The data watcher, a system that can keep track of values (such as current HP), detects changes and automatically synchronizes it with clients
* `IExtendedEntityProperties`, a Forge-provided way of stapling abritrary NBT data to entities, allowing additional stats to be stored by that entity
* References to other entity instances, that being the entity that is currently riding this entity, or the entity that the current entity is riding
* Position and rotation, as well as a method that allows entities to move while respecting block collisions
* An update loop, because most entities actively do things and therefore need logic to run every tick
* A functiona called when attacked, i.e. when using melee on it (assuming it can be collided with), hit by a projetile or inside a damaging area of effect like an explosion
* Methods for loading and saving data to NBT, so that entities can remember values even after unloading

## The many types of entities

Because there's so many types of entities, the class hierarchy is a fair bit deeper than for example on blocks or tile entities. This short overview should hopefully help wrapping your head around it:

* `EntityLivingBase` - makes entities collidable, adds health, attributes, timers for jumping and attacking, head rotations, the combat tracker and potion effects
  * `EntityLiving` - adds various helper instances for AI targeting, pathfinding and looking around, makes the entity leashable and nametaggable
    * `EntityFlying` - used by ghasts, provides movement code that ignores gravity
    * `EntityCreature` - provides a common "wander around" AI task, as well as a simple "follow and attack" AI in `updateEntityActionState()` if AI tasks are disabled
      * `EntityMob` - implements some default player targeting behavior, prevents natural spawning in the daylight, and provides the "attack damage" attribute
      * `EntityAgeable` - makes creatures agable, and using a spawn egg on an existing creature will spawn a child version
        * `EntityAnimal` - allows breeding, this is the base class for things like cows and chickens
          * `EntityTameable` - the base class for ocelots and wolves, keeps track of the owning player and provides a default AI task for sitting and waiting
      * `EntityWaterMob` - the base class for squids, causes the creature to drown when *outside* of water
  * `EntityPlayer` - the base class of the player, contains the inventory, player capabilities, game profile, experience level and currently used item (held right click, like food that is being eaten). Implemented by `EntityPlayerMP`, which is the player class used by the server.
    * `AbstractClientPlayer` - the base class for players as perceived by the client, implemented by `EntityPlayerSP` which is the client's controlled player and `EntityOtherPlayerMP` which is other players being seen by the client
* `EntityThrowable` - the base class for projectiles like snow balls and enchanting bottles, handles collisions with blocks and entities, knows what entity threw it and provides the math for setting the entity's momentum based on direction
* `EntityMinecart` - provides all the logic needed for minecarts, for following rails, being pushable, and colliding with other carts
* `EntityHanging` - cannot move and deletes itself when taking any damage, also does regular checks if connected to a block to persist, used by paintings and item frames

There are many entity classes that do not extend any of these, rather just extending `Entity` directly. This list however should cover most of the important classes.

## Entity AI

AI is a complex and large topic, how exactly AI does things will be the topic for later. For now, let's see how entities can even *have* AI.

The base class `EntityLiving` has a method called `isAIEnabled()`. Contrary to the name, if this method returns `false`, it will simply use an older version of the AI system with a simple loop that does some basic pathfinder and target related things.

Without the AI enabled, the `updateEntityActionState()` method is run in the update loop, which depending on the implementation (or default implementation that comes with the base class) can do basic AI. For example `EntityCreature`, which most mobs inherit from, implements behavior that makes the entity automatically pathfind towards its target, if the `entityToAttack` field is set to another entity, and if close enough, calls `attackEntity(Entity, float)` (which `EntityCreature` does *not* implement). Spiders, for example, do not have any special AI, they just run towards their target to attack, hence they do not use `isAIEnabled()`.

If `isAIEnabled()` returns `true`, then this entity uses the newer AI task system, where an entity can have a number of AI tasks which are processed, even in parallel. AI tasks are ordered with a priority (for example, a zombie shouldn't focus on attacking more than on swimming, which would cause it to sink while busy trying to attack), and separated in "tasks" and "target tasks" lists, where target tasks run before the other tasks.

A creeper's AI task list is ordered as such:

1. `EntityAISwimming` - basic survival skills are important
2. `EntityAICreeperSwell` - if there's already a target close nearby, this is what handles the creeper's fuse
3. `EntityAIAvoidEntity` - the fleeing behavior from ocelots
4. `EntityAIAttackOnCollide` - a bit misleading, this makes it so that this AI tasks's user will run after its target, however creepers do not have any melee attack implemented
5. `EntityAIWander` - if there's no target, wander around idly
6. `EntityAIWatchClosest` - if there's no target, look at the nearest player, that's why creepers like staring at you in creative mode
7. `EntityAILookIdle` - if there's no target and no nearby player, just look around on occasion

And it's AI target task list:

1. `EntityAINearestAttackableTarget` - this sets the creeper's attack target to a nearby player
2. `EntityAIHurtByTarget` - this will read the creeper's revenge target, i.e. the last entity to attack it, and use it as the actual attack target, the entity to run and explode at. Also has an option via `entityCallsForHelp` to modify the attack target of other nearby creepers that are currently idle, making a single mob that's running towards you tell its friends about you, however creepers do not have this option enabled.

## A word on positions and offsets

Entities have position coordinates using the `double` data type, allowing them to exist at a location with sub-block precision. An entity's exact X/Y/Z coordinates are usually at the bottom in the center. In other words, the hitbox of that entity usually stands centered on top of the exact position.

Minecraft also has a value for a Y offset, i.e. how much the hitbox should be shifted up or down. This is only used in a few rare circumstances:

* Falling blocks use an offset equivalent to half their height, causing their position to be situated exactly in the center of the hitbox, rather than at the bottom
* The player entity that is being controlled by the client is offset so that the exact position is at the eye height. You can see this in action by simply enabling bounding box view with F3+B, then going into third person. Due to a bug, the bounding box is rendered without considering the Y offset, causing it to be on top of the player. **This phenomenon only happens on the client. Other player entities do not have such an offset, and neither do any player entitis from the server's point of view.**
