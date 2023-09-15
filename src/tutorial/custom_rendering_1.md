# Custom Rendering Part 1

So far in this series, we have looked at how to work with hotham's existing rendering process.  Let's do a brief overview of that process.

In the shaders folder, there are three sets of shaders that execute to create the frame by frame output we see on screen:

- First of all, a compute shader `shaders/culling.comp` sets the visibility of every primitive based on data passed to the shader about the left and right clip planes and the bounding sphere that has been calculated for each primitive.
- Secondly, a PBR shader for the vertex and fragment stages calculates the position of each vertex based on the view projection and the position of each vertex in globally oriented space.  It calculates the color as RGB and discards alpha channel information, specifying no blend operation, outputing F16 tonemapped data.
- Thirdly, a vertex and fragment shader for the GUI components preserves the alpha channel for these samples and translates the screen coordinates into vulkan coordinates, creating a final color for each pixel by multiplying the input color by the sampled font texture.

When the `engine.update()` function is called, it eventually calls `begin_frame` to prepare openxr for a new frame and to begin recording a command buffer for the frame.  This command buffer is then available as a field within each indexed frame inside the render context.

When a Hotham program calls `draw_gui::draw_gui_system` during the tick loop, it iterates over each panel/UI panel combination in the world and calls paint_gui, which then performs a render pass for each of these panels to render them into the frame buffer.  Later, the `rendering_system` organizes the data from all the primitive meshes in the scene together and then executes the compute shader against this primitive data to determine which of them are visible.  The rendering system executes a begin render pass for the PBR shader, and for the list of primitives that were determined to be visible executes a draw command for each of these.

Finally, when `engine.finish()` is called, the command buffers are executed.  Due to the way color blending and the font texture are set up, along with the memory barrier and dependencies defined in the PBR render and GUI render passes, the grey coverage in the font texture is sampled at the point on the GUI fragment corresponding with the relevant UV coordinate, and is multiplied with the existing color at that coordinate.  Then the blending operation takes place using the `SRC` color fully represented (`vk::BlendFactor` 1) and the `DST` color represented in percentage `1-SRC alpha`.  This effectively means that, using the default operation of `COLOR_OP_ADD`, pixels from the GUI render passes only contribute to the final output in accordance with the alpha value of the source pixel.  Since this source pixel is the pixel from the font texture, the final output has the original color present only to the extent the source pixel (the new one from the font texture) is transparent.

The two pipelines define a set of dependencies like so:

| Dependency Component | PBR Render Pass                                       | GUI renderpass            |
|----------------------|-------------------------------------------------------|---------------------------|
| SRC                  | SUBPASS_EXTERNAL                                      | 0                         |
| DST                  | 0                                                     | SUBPASS_EXTERNAL          |
| SRC MASK             | empty                                                 | MEMORY READ, MEMORY WRITE |
| SRC STAGE            | COLOR_ATTACHMENT_OUTPUT EARLY_FRAGMENT_TESTS          | ALL GRAPHICS              |
| DST MASK             | COLOR_ATTACHMENT_WRITE DEPTH_STENCIL_ATTACHMENT_WRITE | MEMORY READ, MEMORY WRITE |
| DST STAGE            | COLOR_ATTACHMENT_OUTPUT EARLY_FRAGMENT_TESTS          | ALL GRAPHICS              |

This means the following:
1. The first synchronisation scope for the PBR render pass is an external subpass, vis a vis: The GUI renderpass and any other prior relevant pass.
2. The second synchronisation scope for the PBR render pass is itself.
3. There is no particular mask specifying the types of commands the first synchronisation scope refers to
4. The first synchronisation scope's dependency stages being refered to are color attachment output, which is referred to in the Vulkan Specification as "[... specifying] the stage of the pipeline after blending where the final color values are output from the pipeline. This stage includes blending, logic operations, render pass load and store operations for color attachments, render pass multisample resolve operations, and vkCmdClearAttachments."  Similarly, it is also dependent on early fragment tests, which is defined as "the stage of the pipeline where early fragment tests (depth and stencil tests before fragment shading) are performed."
5. The destination mask is color attachment write and depth stencil attachment write bits.  These are defined as "write access to a color, resolve, or depth/stencil resolve attachment during a render pass or via certain render pass load and store operations. Such access occurs in the VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT pipeline stage."
6. The destination stages are the same.
7. Putting all of this together, this restricts output from being written to the color attachment or depth buffers until the entire stage of writing such output on the prior render pass has completed.

This is done to ensure sanity in fragment blending and ensure the different systems write their color output sequentially.

# What constitutes custom rendering?

Effectively, custom rendering constitutes *anything, particularly Vulkan commands or shader modifications which changes the Vulkan rendering pipeline which Hotham uses to display the world.*

Here are a couple of examples of what might be considered custom rendering:
* Rendering custom object types such as quadrics
* Performing color blending operations to change the way Hotham deals with transparency.
* Rendering objects with a different tint/hue depending on whether they are in a special state (eg selected)
* Pre-processing operations, for example to display an overlay on the entire scene such as a HUD.
* Rendering into a texture some sub-set of the world, for example to create a virtual mirror in which the user can see their avatar when standing in front of it.
* Replacing the entire rendering process.

# How does Hotham deal with custom rendering?

As with all systems within Hotham, it is optional to call the rendering system provided.  Hotham sets up a render pass which includes color and depth attachments with multiple sample anti aliasing, which are resolved from four samples down to one in the process of rendering.  It also sets up multiviews, which allows for rendering a slightly different image for each eye.  However, the attachments within the swapchain which it creates by default can be used by other render passes instead, and there is no reason the fixed processes elaborated in the render context discussion earlier need to happen.

A hotham program could choose instead to generate its own pipeline, for example using outlined primitives, or generate its own vertex and/or position buffers from the world without recourse to the existing buffers created by loading in meshes.  Or it could choose to combine the two to use both world based data and other state based data collected in the course of the program.

So hotham takes the approach of allowing the user to have maximum flexibility to take over the rendering process particularly during the command buffer recording phase. By calling or not calling or using different parts of the existing rendering system, different intents can be achieved.

# How does this impact your own approach to custom rendering?

Conversely, this also means the burden is on you as the developer to ensure you work with the existing sub-systems and with Vulkan to achieve your outcome. There are several considerations you will need to make.

1. What objects are visible on screen?  For this, you will most likely want to use the compute shader and primitive cull data buffer described in the prior section.
2. What objects exist within the world or the program state other than those ordinarily recorded within the primitive buffer which you may need to make consideration of in adapting your code to call the compute shader?
3. In what order do the different types of objects within the world need to be drawn to be displayed correctly on screen?  For example, transparent or semi transparennt objects often need to be rendered last and in reverse order of distance from the camera.
4. What pipeline barriers need to be inserted to ensure correct synchronisation?
5. What additional resources do you need to provide to Vulkan in order to ensure correct output?  For example: uniform buffers, textures, lighting information.

# Some general considerations for managing custom rendering.

Make sure you have some flag, struct or other method within your world data to identify objects you want to render differently.

To separate the world objects which need to be treated differently, I recommend the use of marker structs.  An example of this would be abstracting the `Visible {}` marker which the rendering system checks to gather objects prior to sending the data to the compute shader.  Visible objects which are handled by another shader may be marked with an attribute such as `SemiTransparent` or `SpecialObject` to define items that go to one pipeline, multiple pipelines, or none.

Also be aware that in order to achieve many effects, synchronisation is going to be invaluable.  The command `cmd_pipeline_barrier` provides a flexible way of setting up complex dependencies for commands within a render pass, while the render pass definition itself provides a means of identifying dependencies between earlier and later passes.  These dependencies and synchronisation methods are crucial to ensuring correct outcomes without undefined behaviors.

In the remaining parts of this tutorial, we are going to look the general steps such as pipeline creation, culling, memory barriers and swapchain images. We'll also look at how you might structure a change in the rendering to suit a particular use case.  Specifically, we'll take a quick look at color blending.  It is common to want objects to be partially transparent, in order to implement things such as wrist menus, captioning, heads up displays and other billboarding solutions.  This is not as simple as it appears, but we will take the first steps to create a passable output before leaving you to experiment further.  I will be focusing on code in the next section and specifically the use of `ash::vk` to set up the pipeline.
