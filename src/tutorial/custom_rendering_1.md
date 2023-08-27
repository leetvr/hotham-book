# Custom Rendering Part 1

So far in this series, we have looked at how to work with hotham's existing rendering process.  In the shaders folder, there are three sets of shaders that execute to create the frame by frame output we see on screen:

- First of all, a compute shader `shaders/culling.comp` sets the visibility of every primitive based on data passed to the shader about the left and right clip planes and the bounding sphere that has been calculated for each primitive.
- Secondly, a PBR shader for the vertex and fragment stages calculates the position of each vertex based the view projection and the position of each vertex in globally oriented space.  It calculates the color as RGB and discards alpha channel information, specifying no blend operation.
- Thirdly, a vertex and fragment shader for the GUI components preserves the alpha channel for these samples and translates the screen coordinates into vulkan coordinates, creating a final color for each pixel by multiplying the input color by the sampled font texture.

When the `engine.update()` function is called, it eventually calls `begin_frame` to prepare openxr for a new frame and to begin recording a command buffer for the frame.  This command buffer is then available as a field within each indexed frame inside the render context.

When a Hotham program calls `draw_gui::draw_gui_system` during the tick loop, it iterates over each panel/UI panel combination in the world and calls paint_gui, which then performs a render pass for each of these panels to render them into the frame buffer.  Later, the `rendering_system` organizes the data from all the primitive meshes in the scene together and then executes the compute shader against this primitive data to determine which of them are visible.  The rendering system executes a begin render pass for the PBR shader, and for the list of primitives that were determined to be visible executes a draw command for each of these.

Finally, when `engine.finish()` is called, the command buffers are executed.  Due to the way color blending and the font texture are set up, along with the memory barrier and dependencies defined in the PBR render and GUI render passes, the grey coverage in the font texture is sampled at the point on the GUI fragment corresponding with the relevant UV coordinate, and is multiplied with the existing color at that coordinate.  Then the blending operation takes place using the `SRC` color fully represented (`vk::BlendFactor` 1) and the `DST` color represented in percentage `1-SRC alpha`.  This effectively means that, using the default operation of `COLOR_OP_ADD`, pixels from the GUI render passes only contribute to the final output in accordance with the alpha value of the source pixel.  Since this source pixel is the pixel from the font texture, the final output has the original color present only to the extent the source pixel (the new one from the font texture) is transparent.

The two pipelines set up a memory fence like so:

| Dependency Component | PBR Render Pass                                       | GUI renderpass            |
|----------------------|-------------------------------------------------------|---------------------------|
| SRC                  | SUBPASS_EXTERNAL                                      | 0                         |
| DST                  | 0                                                     | SUBPASS_EXTERNAL          |
| SRC MASK             | empty                                                 | MEMORY READ, MEMORY WRITE |
| SRC STAGE            | COLOR_ATTACHMENT_OUTPUT EARLY_FRAGMENT_TESTS          | ALL GRAPHICS              |
| DST MASK             | COLOR_ATTACHMENT_WRITE DEPTH_STENCIL_ATTACHMENT_WRITE | MEMORY READ, MEMORY WRITE |
| DST STAGE            | COLOR_ATTACHMENT_OUTPUT EARLY_FRAGMENT_TESTS          | ALL GRAPHICS              |

This means the following:
1. All external subpasses prior to this one must wait on outputing its color attachments and performing early fragment tests until this subpass finishes writing to its depth stenciil or color attachment.
2. For the GUI renderpass, all read or write operations on all graphic pipeline stages within the renderpass must wait until all prior external subpasses have completed their memory read and write operations.

This is done to ensure the GUI render pass can obtain its input from the PBR renderpass and blend the fragments together in the correct order.  To perform custom rendering that works with the output of these two standard stages, you are most likely going to have to construct a similar memory fence to ensure the color attachments from the primary render subpass have finished their output before using the same color attachments to add further content to the scene.

A consideration in doing so is going to be the order in which you execute your drawing and rendering stages in the tick function.  In most cases, a simplistic dependency such as that presented above for the GUI renderpass is likely the safest option, unless you wish to optimise non dependent stages so they can begin earlier than completion of the prior render passes.

To separate the world objects from one shader from those in another shader, I recommend abstracting the `Visible {}` marker which the rendering system checks to gather objects prior to sending the data to the compute shader.  Visible objects which are handled by another shader may be marked with an attribute such as `SemiTransparent` or `SpecialObject` to define items that go to one pipeline, multiple pipelines, or none.

One way in which this setup could be used efficiently would be in creating custom wrist menus.  Suppose I create an empty texture attached to a mesh plane, within which is a circular shape upon which I draw text dependent on game state, as well as various controls which are also based on the existing state. Around the circular shape, the remaining pixels are fully transparent (0.0 for all RGBA). By using a custom pipeline to blend these objects with the surrounding PBR rendered objects after all other objects have finished drawing, and setting appropriate colliders, as well as parenting the mesh plane to some relevant object such as the left/right hand, you can create a hybrid object which is a combination of static game content and dynamically generated output.

A simple way to do this would be to create a set of shaders which are at first identical to the PBR shaders.  By simplifying the fragment shader to remove aspects of the PBR color calculations you don't require (eg the debug output), and by preserving the alpha channel value from the base color by explicitly setting it after the tonemapping is completed, such a simplistic change in the existing PBR renderer code can be used in conjunction with the color blending options specified at pipeline creation to acheive the desired effect.

In the next and final section, we will look at some sample code to accomplish this.
