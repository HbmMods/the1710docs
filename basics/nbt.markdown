---
layout: page
title: Named Binary Tags (NBT)
permalink: /nbt/
---

# /Named Binary Tags (NBT)

NBT is a way for things in Minecraft to store information. This of course is a very broad and vague description, but that's because NBT can do a broad number of things.

NBT is used for example to store information about entities and tile entities once they unload, all their values are serialized using NBT, saved to disk, and once loaded again, deserialized.

Everything related to NBT stems from a class named `NBTBase`. The class itself doesn't do much except say "look, this is NBT serializable" and provide a few utilities like turning NBT types (more on that in a second) into integer IDs and back which is used internally for turning NBT data into raw bytes on the disk and back. Classes that extend `NBTBase` can be put into three categories: **NBT primitives**, which store actual values, **NBT collections** , which encompass multiple NBT primitives, and **NBT special tags**, of which there is only one, that being `NBTTagEnd` which signifies the end of a byte stream.

NBT primitives are rarely used directly, but it's still useful to know which ones exist. Most data types are covered, like integers (and variants), floating point numbers, strings and even int and byte arrays. NBT primitives are mainly just pieces for NBT collections which are the kind of NBT you'll most commonly work with.

NBT collections come in two flavors, the `NBTTagList`, which is a dynamic sized list that can hold one type of NBT data (primitives or either NBT collections), and `NBTTagCompound`, the most commonly used form, which is what gives NBT its name, as it allows the naming of tags. In essence, a tag compound is like a hash map, with the key type being strings. This makes serializing and deserializing data very easy, since there's no fixed order in which these things need to be done, and tags being absent or serialized but then never used doesn't cause any issues either (fundamentally at least, your implementation might still break).

Tag compounds are not only used in saving entity and tile entity data, but also for extended properties, a Forge-provided utility for dynamically stapling extra data to entities that otherwise would not have the fields to store such data (hence why the named tag part of the system is so handy, you can add things arbitrarily), added info for item stacks, which allows for stack-based data beyond the single 16 bit integer that is metadata (or "damage", what tools use for durability).

Most of the handling of NBT data, that being saving to and writing from disk, is already taken care of by vanilla, so it's not something you will often encounter. When it does become necessary, there's the `CompressedStreamTools` class in the `net.minecraft.nbt` package, which contains handy util methods for converting NBT data between `InputStream` as well as `OutputStream`.

In addition to creating streams which usually just end up being bytes on a disk, NBT data can also be (de)serialized as text. For example, the NBT data that can be added to items using hte `/give` command uses this exact method. Using `toString()` on any NBT instance will create valid JSON-parsable strings, which for NBT collections also contains any child items, including nested collections. Inversely, the `JsonToNBT` class provides the static utility method `func_150315_a` (which Forge never deobfuscated...this is going to become a trend, get used to that) which will try to parse a JSON string into a `NBTBase` instance.
