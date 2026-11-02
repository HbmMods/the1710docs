---
layout: page
title: Rendering
permalink: /rendering/
---

# /Rendering

Requires:
* [How Minecraft runs](/the1710docs/minecraft)

Rendering is what actually puts pixels on the screen, i.e. "makes things show up". Anything we want to see in the game needs to be rendered. It doesn't matter whether it's an NPC, a block, a tile entity, or the clouds in the sky, all rendering in Minecraft follows roughly the same principles.

Rendering is done using a library called [OpenGL](https://www.opengl.org/), 1.7.10 uses GL version 11 (and sometimes 12) for its rendering. If you have prior experience with GL, for example from a C++ application, the same knowledge will still largely apply.

Without concrete examples, most of the info on this page will appear next to worthless. Rendering's difficult like that. Try to remember a few things before checking out the examples section.

## The Fundamentals
There's things like frame buffers and stuff like that. I don't know anything about that. Minecraft does some setup stuff involving that.

## The Client Tick Loop
Every client tick, which is equivalent to a *frame*, the game will try to render everyhing that's seen. The skybox, the blocks, GUI screens, the HUD, held items, entities, and so on. In order, it will:

* Update the "light map" which is some math which sets the current brightness depending on the time of day, as well as shadow colors, which can differ depending on the current dimension. Ever wondered why light level 0 in the nether looks brighter in the nether than it does in the overworld? Because it actually is, the nether's light map actually has a higher minimum value.
* Handle the mouse input, which is part of the the `EntityRenderer.updateCameraAndRender(float)` function, but does not fall into the scope of rendering
* Set up the fog color which can depend on the biome, dimension and depth
* Set up the camera transform, i.e. rotates the view based on the client player's rotation, and handles things like the tilt caused when taking damage
* Render the skybox, which despite looking really far away is actually only 100 units in each direction, it's just that everything that renders afterwards does not account for its depth, and therefore always in front of the skybox even when further away
* Try to add nearby chunks that are loaded but do not yet render to the "please render this" list
* Render clouds
* Sorts and then renders all chunks from the "please render this" list and renders them using "pass 0", which only includes blocks with no semi-transparency (i.e. excluding water and stained glass)
* Render entities (pass 0)
* Render the little wireframe selection when aiming at a block
* Render the little broken block overlay when mining something
* Render all particles
* Render the weather overlay, i.e. rain or snow
* Renders all chunks again, but this time using "pass 1" which only includes semi-transparent blocks. Rendering semi-transparent blocks near the end works around a fundamental issue regarding semi-transparent rendering and makes them show up correctly most of the time (transparency is surprisingly hard).
* Renders entities again, this time pass 1
* Render clouds *again*, but only when the view is above Y:128 which is the cloud ceiling. This is noticeable when going above the clouds, usually there's a visible "jump" where the clouds suddenly seem to become less transparent, this is due to the clouds rendering a second time *after* almost everything else*.
* Render the player's arm when in first person
* Render the HUD, including the item bar, all the gauges, the nether portal overlay and the crosshair
* Render the currently open GUI, if there is one

... and this about covers everything you can actually *see* in the game. If you can name it, it's probably in there.

## Blocks vs. (Tile) Entities
There is an important distinction to make with how certain things are rendered. Things like entities and tile entities are not just rendered but also *composed* every frame, i.e. vertices are placed, transformations applied, and then rendered. With some fancy optimizations (display lists, VBOs), the vertex part can be optimized away, but the transformations can still apply, that's why for example tile entities are commonly used for blocks that use more complex animations, like chests with their moving lids and beacons with their rotating beams.

Blocks on the other hand are only composed whenever the chunk requires it, either if a block is placed, removed, changed, or the block light changes (that being light sources, but not counting the change in light level form the time of day, the lightmap does that). Therefore, standard block rendering is not suited for fluid animations, the only thing that Minecraft provides is animated textures, for example the flowing texture of water. Blocks being rendered this way has the advantage that we can do many blocks at once with a single render call instead of having to do hundreds of render calls per chunk, which greatly improves performance. Since we can expect chunks to stay as they are over a timespan, chunks are cached as a display list, which is a horrendously dated and not very efficient (but still more efficient than not doing it at all) way of sending the render data to the GPU and keeping it there for a while instead of having to send it over and over every tick. Modern versions and many performance mods make use of VBOs, which are conceptually similar but more modern and more performant too.

## Transformations
An important part of rendering is the transformations, which include operations such as scaling, translating ("moving") and rotating. Once a transformation is defined, it will apply to everything that is rendered past that point. If we want to render a thing that's rotated 45° along the Y-axis, we can do it as such:

```java
GL11.glRotated(45, 0D, 1D, 0D);
// render our thing here
```

For rotating, the first number is the amount by which we want to rotate it, and the next three numbers are the axis by which we rotate (usually it's always 0 or 1 and only one of the three axis is 1 for clarity).

However, in our example, everything rendered past that point will *also* be rotated by 45°. For this, we can use the GL push and pop matrix commands. With these, we can define "zones", pushing a matrix starts a zone and popping it ends it, which undoes all transforms done since the last push:

```java
// render something at no angle

GL11.glPushMatrix();
GL11.glRotated(45, 0D, 1D, 0D);
// render our thing here
GL11.glPopMatrix();

// render something else at no angle
```

For clarity, it can be helpful to add brackets to these "sections". It is also important to pop after every push operation, as pushing or popping too many times will cause the famed "buffer over/underflow" errors.

```java
// render something at no angle

GL11.glPushMatrix(); {
    GL11.glRotated(45, 0D, 1D, 0D);
    // render our thing here, and now it's more apparent it's in a different section
} GL11.glPopMatrix();

// render something else at no angle
```

Transformations happen around the origin point, which is relevant for scale and rotate operations. Translating something will also translate the origin point, this is useful for defining exactly where our rotation point should be defined:

```java

GL11.glTranslated(0, 1, 0); // this moves the origin up by one unit
GL11.glRotated(45, 1D, 0D, 0D); // rotate around the X-axis
GL11.glTranslated(0, -1, 0); // manually undo the origin change to prevent our object from floating

// render something here, now rotated around an axis 1 block above
```

The way transforms chain together may not be entirely intuitive at first, but once you try to make something with a lot of layered transforms (like a robot arm where each joint adds more rotation), it will become apparent that the system does make a lot of sense the way it is.

## How to actually render things
Imagine a cube. A cube is six sides, with each side being a square. When it comes to defining these things, OpenGL gives us multiple options, for example we can render only three vertices as triangles, or we can use a render mode that automatically turns four vertices into a square, which is actually just two triangles in a trench coat.

For rendering things out of straight vertices Minecraft provides the `Tessellator`, a useful (if brittle) utility that handles some things for us. Let's render a simple square with no texture:

First we want to actually grab the tessellator's instance. The tessellator is a singleton of which only one exists, and we "borrow" it for as long as we need it. Then we begin a square drawing operation:

```java
Tessellator tess = Tessellator.instance;
tess.startDrawingQuads();
```

Now we want to define our vertices, i.e. the corners of the square:

```java
tess.addVertex(-1, 0, -1);
tess.addVertex(-1, 0, 1);
tess.addVertex(1, 0, 1);
tess.addVertex(1, 0, -1);
```
Usually there are a few more things involved, but for this example we can neglect that. Finally, once we are done, we tell the tessellator to actually draw:

```java
tess.draw();
```

`startDrawingQuads()` in this example is just one of the functions the tessellator provides. It also has a generic `startDrawing(int)` method where we can pass our own drawing mode. A common one is drawing triangles, which would look like this:

```java
tess.startDrawing(GL11.GL_TRIANGLES);
tess.addVertex(-1, 0, -1);
tess.addVertex(-1, 0, 1);
tess.addVertex(1, 0, 1);
tess.draw();
```

In general, draw operations are expensive. We want to pack as much vertex data in every draw call as we can. In our cube example, we want to define 24 vertices before we draw, rather than doing six separate draw calls. Depending on the drawing mode, the rendering will automatically group together vertices to faces, using quad rendering, that's every group of four successive vertices.

Example:
```java
tess.startDrawingQuads();
// first face
tess.addVertex(-1, 0, -1);
tess.addVertex(-1, 0, 1);
tess.addVertex(1, 0, 1);
tess.addVertex(1, 0, -1);
// second face
tess.addVertex(-1, 1, -1);
tess.addVertex(-1, 1, 1);
tess.addVertex(1, 1, 1);
tess.addVertex(1, 1, -1);
// third face
tess.addVertex(-1, 2, -1);
tess.addVertex(-1, 2, 1);
tess.addVertex(1, 2, 1);
tess.addVertex(1, 2, -1);
tess.draw();
```

...will create three squares floating above each other. As explained earlier, vertices are grouped together based on the order and the draw mode used, here successive groups of four vertices are combined into distinct squares. If we were to take the last vertex definition and move it to the top, the order would break, and all squares would look messed up, with parts connecting that shouldn't.

Nonetheless, having separate draw calls is often necessary. Transforms will only apply to entire draw calls, so if we want to transform parts differently, we have to separate them. For example:

```java
tess.startDrawingQuads();
// we define a few vertices here
tess.draw();

GL11.glRotated(spinny, 0D, 1D, 0D); // "spinny" is a variable that changes every tick

tess.startDrawingQuads();
// we define more vertices for the object that should spin
tess.draw();
```

This principle not only applies to transforms, but to any change in the render state, like lighting, transparency (via blend modes), face culling settings and so on.

Finally, when defining vertices, order does matter. When face culling is active, faces will only render on one side, under the assumption that the other is not visible, which requires an "inside" and an "outside". The way this works in Minecraft's rendering is by simply checking whether the face was defined clockwise or counterclockwise and only rendering one side based on that (sort of like how the left hand rule works). I forgot whether clockwise or counterclockwise definitions show the face, if it's not one then it's the other. Quick tip for fixing orientation: If the face is backwards and it's a quad, simply swap the first and the second vertice, and then the third and fourth, this will flip the rendered side. If it's a triangle, simply reverse the order of the vertex definition.

## Normals and UV

The previous examples only used vertices. This works for shapes with neither textures nor lighting, but nothing else.

Normals are vectors that point "away" from a face in order to tell it what orientation it has. The system can't figure it out on its own, since that would be too computationally expensive to do, and there's certain use cases for supplying normal definitions that aren't actually correct. The normals are primarily used for lighting, in Minecraft, lighting is relatively simple, faces that point upwards (i.e. normals with a large Y component) are fully lit and faces that point downwards (negative Y component) become dark.

Billboarded objects like particles which have a normal vector that points to the player often times have their normals set up in a way that they point up, which makes them fully lit.

In addition to lighting, most objects that are rendered also have a texture. For this, we first have to *bind* a texture, which is typically done via Minecraft's texture manager:

```java
Minecraft.getMinecraft().getTextureManager().bindTexture(tex); // tex is an instanceof ResourceLocation that points to our texture path
```

... and then we have to define the UV. "UV" is not an acronym, it is simply the name of two axis, like XYZ are for three dimensional space, only that UV comes from the two dimensions of a flat texture. In essence, we have to tell each vertex, which exists in 3D space, where it maps to the texture in 2D space. For simple block models this is simple, as the UV of each vertex of a side is just one corner of the entire texture:


```java
Tessellator tess = Tessellator.instance;
tess.setNormal(0F, 1F, 0F);
tess.startDrawingQuads();
tess.addVertexWithUV(-1, 0, -1, 0D, 0D);
tess.addVertexWithUV(-1, 0, 1, 0D, 1D);
tess.addVertexWithUV(1, 0, 1, 1D, 1D);
tess.addVertexWithUV(1, 0, -1, 1D, 0D);
tess.draw();
```

`U` is typically the horizontal and `V` the vertical axis on the texture. On our second vertex, `0D, 1D` basically means "bottom left corner", as the coordinate origin is the top left, and `U` is 0, not going horizontal, and `V` being 1, meaning going down vertically 100% of the way of the texture's height. Values outside of the [0;1] range are also valid, in most cases the texture will simply loop. Do note that this does not work on stitched textures like the block and item atlas, since going "out of bounds" means accessing a texture of another block/item.

In our example, we also defined a normal vector that has a Y component of 1, i.e. going straight up.

## Capabilities
The rendering system also allows toggling certain capabilities, i.e. "settings" that change how things are rendered. Capabilities do not follow the "push-pop" rules of transformations, and will "leak" if toggled and not reverted, applying unintended effects to other objects.

Common capabilities include:

* `GL11.GL_LIGHTING` which toggles normal-based lighting (no lighting does NOT mean fullbright, since the total shade can still be modified by the lightmap)
* `GL11.GL_CULL_FACE` which toggles face culling, i.e. that faces should only render on one side to save on resources. When rendering "thin" objects with no actual volume, face culling either needs to be disabled, or a second face with the opposite direction has to be rendered. The latter option is usually preferable, since un-culled faces have the same lighting on both sides, which looks terrible especially on horizontal faces. If the two faces option is chosen, make sure the faces are culled, since having two un-culled opposing faces that occupy the same space will cause z-fighting (the flickering effect when two things are stuck in the same space).
* `GL11.GL_BLEND` is what allows semi-transparent things to be rendered. Transparency brings its own issues, though, and there's multiple transparency modes that need to be configured for blending to be correct. Always make sure to disable blending in contexts like tile entities, as blending is not the expected default, and will cause a state leak where other unrelated tile entities will render semi transparent.
* `GL11.GL_TEXTURE_2D` which if disabled, causes the texture to not be used, making the object to be rendered white, unless a color is set for the current context. This also means UV info does not apply here, so if no texture is supposed to be used in this context, defining UV can be skipped.

There are, of course, many more capabilities as well as "attributes", a system that I barely understand myself. A more in-depth look at how exactly all these things are used will be included in the examples later on.

