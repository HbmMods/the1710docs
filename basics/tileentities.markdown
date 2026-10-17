---
layout: page
title: Tile Entities
permalink: /tileentities/
---

# /Tile Entities

Requires:
* [Client/Server Concept](/the1710docs/clientserver)
* [Blocks](/the1710docs/blocks)
* [Named Binary Tags (NBT)](/the1710docs/nbt)

Tile entities are objects that accompany certain blocks within the block grid. Tile entities allow blocks to store way more data than what would be possible with just their metadata ranging from 0-15. Containers such as chests and machines such as furnaces and beacons all use tile entities for holding items and receiving update ticks.

Tile entities are invisible by default, but they can have a tile entity special renderer (TESR) attached to them, allowing them to be seen in the world, making it possible to give blocks animations that otherwise would not be possible with standard block rendering. Chests for example are rendered as TESRs, which is the reason why the block break particles don't show up on them. Chests use TESRs to animate the lid, enchantment tables for the floating book, beacons for the light beam, signs for their text, and so on.

Tile entities are created automatically by blocks that use the `ITileEntityProvider` interface, which has the `createTileEntity(World, int)` method, allowing a tile entity to be created depending on the block's metadata. However, removal of tile entities after a block has been broken, does not happen automatically. Vanilla Minecraft already has an abstract base class for handling that, being `BlockContainer`. You will likely get by just fine if you use `BlockContainer` as a base class for blocks that need tile entities.

Tile entities don't provide a lot methods to work with, but the things they do provide are incredibly powerful:

* Methods for loading and saving NBT data. This allows tile entities to store arbitrary data, like settings, counters, item stacks and anything you can serialize into NBT data.
* An update method which is called every tick in the world. This is the meat and potatoes of any machine that does recipes automatically, handles items in some way, or interacts with blocks.
* A built-in system for receiving update packets which vanilla provides, allowing server-side changes to be synced to clients easily
* Methods called when the tile entity is invalidated (i.e. the owning block is broken and the tile subsequently destroyed) or unloaded (via chunk unload)
* `markDirty()`, which tells the tile entity that it has unsaved changes and that it needs to save to the chunk upon unload. Tile entities do not save their changes unless this method is called. This also triggers a block update to neighbors.
* A render bounding box and a max render distance to cull tile entity renderers that aren't visible, assuming this tile has a renderer
