---
layout: page
title: Items and Item Stacks
permalink: /items/
---

# /Items and Item Stacks

We are now finally talking about actual in-game content. Hooray!

First we have to ask, what is an item, what is an item stack?

In the context of modding the game, an item is specifically an item *type*. Imagine you have an inventory full of diamonds. Each diamond is in fact the same diamond *item*. If you ask a diamond item which concrete diamond it is, it can't know that, because it's *every* diamond.

An item stack is an item that's in a slot, i.e. in your inventory, in a chest, held by a zombie or even dropped in the form of an item entity. The `ItemStack` class is a wrapper that contains an **item instance**, the **metadata** (while an integer, it gets truncated to a short when saving, so that ranges from -32,768 to 32,767) which is usually used to either denote an item subvariant (like golden apple tiers or coal/charcoal) or tool durability, **optional NBT data** (for more complex stored data which can't be covered by the meta, for example enchantments or custom names), and a bunch of utility methods for handling and comparing item stacks.

## About Items

So now we are talking about items, what can they do?

* Each individual item has a unique name which is used by the registry, names need to be unique of course, while the actual item class can repeat (in fact, item with no special behavior just use the `Item` class).
* Items can define many traits, however, most getters for said traits have the relevant item stack this item is from as a parameter, allowing for better stack data aware customizability:
  * The maximum stack size of that item
  * The maximum durability, if it can be damaged
  * The unlocalized name, i.e. what's used for the translation of the item name, can also be adjusted based on subtype
  * Item tooltips
  * The item's icon
  * A container item, i.e. another item that is left behind when using this item for crafting (like the bucket when using a bucket of water in crafting)
  * Whether the item has an enchantment glint or a colored name
  * Whether the item is a tool, what harvest level it has and what kind of tool it is
  * What creative tab it's in, and what variants are listed in the tab
* Items also have many methods which are called in certain circumstances, allowing them to be functional beyond being crafting items:
  * Methods for when the item is right clicked in the air, used on a block, or used to hit an entity
  * Methods called every player tick as long as the item is in the inventory, or equipped as armor
  * Methods for altering the item entity when dropped such as max lifespan, extra events on entity update, or replacing the item entity entirely
  * Extra behavior for when the item is eaten, assuming it's item use action is "eat", allowing it to be eaten
* The aforementioned lists are non-exhaustive, items have many traits and functions that we will explain in more detail later.

## Item Icons

By default, items are rendered as 2D sprites from a texture. Using the `setTextureName(String)` method and then setting the texture name for that item works just fine for basic items, however when dealing with subtypes, you will have to work with icons.

Icons are just instances that hold coordinates on the texture atlas. When textures are loaded, all textures registered to the icon register are stitched together into a big sheet (the item texture atlas), and the icon instance held by the item is told the coordinates of where the texture is on that sheet.

For most purposes, it's sufficient to make a field (or an array, but do sanitize the indices!) for every texture of an item variant, then instantiate them by using the item registry provided by `registerIcons(IIconRegister)` (simply call `registerIcon(String)` with the texture name and it'll spit out the icon instance), and then return the correct icon instance based on the stack's metadata in `getIconFromDamage(int)`. There's a few additional variants of this method with varying parameters including the entire item stack instance (allowing you to make NBT-aware textures) and the render pass (for multi-layer textures, for example in vanilla the leather armor which is dyeable).
