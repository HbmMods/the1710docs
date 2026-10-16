---
layout: page
title: Blocks
permalink: /blocks/
---

# /Blocks

Similar to items, a block instance essentially only describes the block's *type*. Each dirt block in the world is actually the same dirt block instance, it would be way too RAM-consuming to have an instance for every single individual block in the block grid. Each position a dirt block is at simply points to the same block instance. Like with items, the block instance does not know which exact concrete block it is, because it is *all of them*.

The block grid not only contains blocks but also metadata, similar to how items have metadata. However, due to budget cuts, block meta only ranges from 0 to 15, so each block can only have 16 subvariants. Wool for example is just one block, but each color is a different subvariant, and there's exactly 16 colors of wool.

Just like items, block textures, by default, are also handled via icons. There's a convenient `setBlockTextureName(String)` method allowing the block's texture to be set, however when dealing with multiple subvariants with different textures, we once again need to work with icons directly. The way it's done is near identical to items, make fields for holding the icons, use the icon registry in the `registerBlockIcons(IIconRegister)` method to create new icon instances from the texture names, then provide the icon using `getIcon(int side, int meta)`. Once again, there's additional variants for that method with different parameters, like one that skips the meta parameter entirely, and one that supplies the world instance along with coordinates, so we can do things like neighbor checks.

When setting up a block, we can configure things like:

* The registry name, which is unique to each registered block instance
* The creative tab this block is listed in
* The block's hardness (mining resistance) and explosion resistance
* The sound category used for this block when placing/breaking/stepping on it
* The block's material, used in various checks as well as for the color of this block used by the map item
* The light level this block emits
* The occupancy, i.e. whether this block blocks light, and how much
* The block's render type, which lets us use most of the vanilla block shapes, as well as our own if we register an ISBRH
* The block's collisions and hitboxes
* Whether this block receives random update ticks, like crops that grow
* The item this block drops, the amount and the metadata, as well as an alternate method for defining an entire list of dropped item stacks

In addition, blocks let us implement methods for giving them functionality:

* What happens on an update tick, either random or scheduled
* What happens on a random client-only display tick, which is what furnaces use to spawn flame particles
* What happens when the block is broken, placed, or a neighbor has changed (block update)
* Actions when the block is right clicked, hit, walked on or collided with

You may rightfully think that the number of things we can do with a block are quite limited, and that only having a single number from 0-15 to work with doesn't let us do a whole lot. You can't really store an item stack inside a number that small, let alone 27. For that purpose, there's something called a **tile entity**. Tile entities don't have much in common with regular entities, they don't even share the same base class. Tile entities can be provided by blocks and are (by default at least) invisible constructs that also exist within the block grid. Tile entities can contain arbitrary data, which is saved using NBT, and they can also receive regular update ticks. Most blocks that hold items or perform a task over time with tick precision use tile entities (a notable exception being the Ender Chest, which while it does use a tile entity for rendering and the lid animation, actually stores the items within the player data).

Tile entities are a large and important topic, so we will go into detail in the next chapter.
