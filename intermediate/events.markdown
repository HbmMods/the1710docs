---
layout: page
title: Forge Events
permalink: /events/
---

# /Forge Events

Requires:

* [Client/Server Concept](/the1710docs/clientserver)

When making a basic mod, most things are handled via **registries**, in essence just collections of instances, for example blocks and items, that are handed to the game to use. This allows us to add a lot of new things, but how would we extend existing contents? How could we even *modify* some of it?

Every so often though the code, you may see something relating to *events*. Forge has injected code into many places to "post" new instances of various events, for example in the base constructor of `Entity`:

```java

MinecraftForge.EVENT_BUS.post(new EntityEvent.EntityConstructing(this));
```

There are a few parts to understanding what this actually does and how we can make use of it. Let's start from the back and work our way outwards.

`EntityConstructing` is a type of event. Events need a type so we can tell where this event is coming from, i.e. what part of the code is producing this event, and in order to supply additional values as context. For example, most entity events contain a reference to the `Entity` instance this event is related to.

This event is *posted*. This means that Forge's event system will go over a list of *event handlers*, i.e. methods made to process this event, and call those methods, using the event instance as the parameter. This means that, while vanilla is doing things, we can inject our own code whenever there's an event.

`MinecraftForge.EVENT_BUS` is the event bus to use for this event. An event bus is basically just a "group" of events, an event is only pushed to one event bus, and that bus will only consider event handlers that are registered to said bus. Forge provides several event buses for grouping events:

* `MinecraftForge.EVENT_BUS` handles most gameplay events not handled elsewhere
* `MinecraftForge.TERRAIN_GEN_BUS` handles events related to terrain generation
* `MinecraftForge.ORE_GEN_BUS` handles only the events produced when generating ore veins
* `FMLCommonHandler.instance().bus()` handles Forge-internal events as well as things like ticking (of the world, the client and entities), login, logout, and key/mouse input

It is not necessary to memorize these, when registering an event handler, simply look up to which bus the desired event is posted, and then attach your event handler to that bus.

Now to the part that actually lets us do things: **The Event Handler.**

The event handler is just a class with methods that usually aren't directly invoked anywhere, since the event bus does that for us. In order to tell the event bus that a method should be used for a certain event, we need to do the following things:

1. Our event handler needs to be registered by using the bus' `register(Object)` method. There are two ways of doing this, we can either use the class directly (like `MyHandler.class`), which requires all event handling methods to be `static`, or we can make an instance of the handler, in which case the methods should *never* be static. (Note: I *heard* this is how it works, I have never actually tried passing the class and using static methods. In case of doubt, just make an instance of your handler.)
2. Our handling method needs the exact class used for posting this event as the sole parameter. All context for the event comes from the event instance itself, there's no extra parameters, because that's all we get from the event bus.
3. Our handling method requires the `@SubscribeEvent` annotation. This is what actually tells the system, "hey this is an event!", after which the event handling system autmatically assigns the method to the event as determined by the parameter type.

## Example

`EntityLivingBase` has a method called `jump()` which handles exactly what you think it does. This method calls `ForgeHooks.onLivingJump(this)` which posts an event of the type `LivingJumpEvent` to `MinecraftForge.EVENT_BUS`. This means that for this example, our event handler at the very least has to be registered to `MinecraftForge.EVENT_BUS`.

`LivingJumpEvent`'s context is `EntityLivingBase entityLiving`, which is just the entity that is jumping. Since the entity instance contains references to the world instance as well as the entity's position, velocity, and all other related values, this is actually a lot of context to go off of. If we check the code for `EntityLivingBase.jump()`, we can see that the event is posted *after* evertyhing else. It is important when exactly events are posted, because in many situations this can make a difference. Here for example, the entity's Y-motion has already been changed by 0.42. Let's see what an event can look like:

```java

@SubscribeEvent
public void onEntityJump(LivingJumpEvent event) {
    if(!worldObj.isRemote) {
        event.entityLiving.setFire(10);
    }
}
```

Since `jump()` is part of `onLivingUpdate()` which itself is called by `onUpdate()` with no side-checks, jumping will run on both client and server. In addition, being on fire is something that happens serverside and is automatically synchronized to clients, so it is not necessary to do on clients, hence why here we have to check for `!worldObj.isRemote` to ensure the entity is only set on fire on the server. And like this, we have hooked our own auto-ignition code into the entity's jumping code.

Of course, there's more useful things to do with events, which at first you may not even realize. For this, it's important to know what types of events there are.

## Canceling Events

Certain events can be canceled, in which case that event's `isCancelable()` returns `true`. If an event is canceled, usually the event bus that this event got posted to will return `true`, and the subsequent code will be skipped. This of course always depends on the exact event you're trying to cancel, so it's important to look up the source code of where the event is posted to know what exact consequences canceling the event has.

For example, `GuiIngameForge`, Forge's replacement for the HUD class, has many events that are posted when the HUD is rendered, which can be canceled. Each HUD component triggers to events, a `RenderGameOverlayEvent.Pre` with the HUD elements type attached which can be canceled, preventing the type from being rendered, allowing us to disable HUD elements (and potentially replace them with our own), and a `RenderGameOverlayEvent.Post` which cannot be canceled, which only gets posted after the HUD element is drawn (and consequently, is never triggered if the `Pre` event was already canceled).

Once an event is canceled, by default, all subsequent event handlers are skipped and will not receive the pushed event, unless the `@SubscribeEvent` annotation has the `receiveCanceled = true` flag set. This can be useful in situations where we want to run certain code no matter what, even in situations where we can expect other mods (or ourselves) to cancel events.

Another example: We want to draw a new HUD element. It's easiest to just render it as part of an existing element, so we just use some element's `Pre` event and render it there, since the `Pre` event is always guaranteed to run (unless the `ALL` type got canceled, disabling the entire HUD). However, HUD elements are a somewhat common thing to cancel, since some mods implement their own replacement, and canceling the event will prevent our extra part to render. In this case, we use `receiveCanceled`.

## Results

Events can produce a `Result`. Only a few events do so, and the concept is similar to event cancellation, but it also has an explicit "allow" setting.

* `Result.DEFAULT` is the default result (duh) and does not do any special logic.
* `Result.DENY` explicitly disallows this action. Zombies can post the `SummonAidEvent` which checks if a zombie can spawn reinforcements (an obscure feature, only on hard mode and only for 10% of zombies), if `DENY` is the result then the check will end and reinforcements are never spawned.
* `Result.ALLOW` does weird things. While it looks like it's intended to explicitly allow certain actions, in most implementations, it pretends like an action has taken place (for example, bone meal is used up and the particles are spawned) without actually doing anything.

Before dabbling with results, it's always important to look up the code where the event is posted to see what each result really does.

## Priority

Sometimes, it only makes logical sense to run an event before all other events, even if other mods may do something different. Inversely, sometimes it only makes sense to run an event after everyone else is done doing their changes. In this case, the `@SubscribeEvent` annotation can have its `priority` set to one of five `EventPriority`s. When an event is posted, all handlers that use `EventPriority.HIGHEST` are run first, then `HIGH`, then `NORMAL` (the default), `LOW`, and finally `LOWEST`.

When canceling an event to ensure something absolutely does not happen and other mods ideally don't do other changes because that wouldn't make sense, it's wise to use `HIGH` or even `HIGHEST` to ensure the event being canceled always happens first.

## Types of Events

There are many events in many places, almost too many to list. Here's a few important ones that you might need:

* `PlayerLoggedInEvent`
* `ClientDisconnectionFromServerEvent`
* `LivingDeathEvent` - can be canceled, preventing death
* `EntityJoinWorldEvent` - when an entity is spawned, can be canceled to prevent spawn
* `LivingUpdateEvent` - every tick for an `EntityLivingBase`
* `WorldTickEvent` - happens twice per tick with different `Phase`, being `START` (before world tick) and `END` (after world tick)
* `LivingAttackEvent` - can be canceled to completely nullify not just damage but also knockback, the damage however cannot be modified here
* `LivingHurtEvent` - cannot be canceled, knockback is inevitable, however the damage taken can be changed
* `ItemCraftedEvent` - when an item is taken out of the crafting grid, ideal for triggering achievements
* `PlayerInteractEvent` - when the player right clicks a block or the air, or left clicks a block
* `RenderGameOverlayEvent.Pre` - before the HUD is rendered, can be canceled to disable HUD elements
* `ItemTooltipEvent` - creates the list of strings used in the item's tooltip, allows us to add custom info to any item
* `ClientTickEvent` - runs every frame, has `START` and `END` phases
* `RenderWorldLastEvent` - at the end of the client tick during world render
* `RenderLivingEvent` - the `RenderLivingEvent.Pre` variant can be canceled to disable rendering for this entity
* `TextureStitchEvent` - when the texture atlas is created, allows us to register `IIcon` instances which are not part of any block or item
* `GuiOpenEvent` - posted every time the open GUI screen changes, can be canceled or have the GUI changed in order to replace GUIs entirely
* `WorldEvent.Load` - when the world starts
* `WorldEvent.Unload` - when the world stops
* `WorldEvent.Save` - when the world saves, which happens regularly or when a singleplayer game is paused
* `DrawBlockHighlightEvent` - responsible for the thin grey frame around blocks that are being aimed at, can be canceled
