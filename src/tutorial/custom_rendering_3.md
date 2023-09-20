# Custom Rendering Part 3

Once you have a working pipeline, you need to use it.  This means:
- beginning a render pass
- sending bind calls for the pipeline, vertex buffers, index buffers, descriptor sets 
- making draw calls into the command buffer after the pipeline bind point to begin sending the data to the GPU.

In this series we will not be covering the trivial case of the only custom rendering change being a change in the
default shaders.  If you are doing this, all you will need to do is consider the implications of your changes on
the rendering and nothing more.

We previously mentioned several examples of custom rendering.  Most versions of custom rendering which are non trivial
will involve changes to the primary render pass conducted by the rendering system. The two possible paths your custom
rendering may take are:
1. Execute your render pass prior to the PBR render pass with appropriate dependencies set up to ensure the output of
your own render pass is taken into consideration by the PBR render pass.
2. Alter or remove the PBR render pass, replacing it with one which selectively renders what you want it to render.
Execute further render passes before and/or afterwards if necessary to render information into other images.

In both of these cases, but more so with case number two, you may need to make use of the methods or objects which the 
existing render pass uses.  Specifically, you may need to:
- Render into the existing acquired swap chain images.
- Perform culling using the compute shader
- Ensure commands run in a specific order.

This section will focus on these tasks.  Using the compute shader has already been covered under the Render Context
page and will be elaborated on again later.  Let's start with rendering into the swapchain images.

# Swapchain Images

During the `engine.finish()` function, the render context is first called to end the frame.  This ends the command
buffer and submits the finished commands.  It then adjusts the frame index, and calls OpenXR to stream the images
to the device using a Frame stream.

Similarly, when a frame is begun, the XR context `begin_frame` is called, which then uses Open XR's `FrameWaiter`
construct to wait on the correct time to begin rendering, then calls begin on the `frame_stream` object.  If the
frame waiter advises that rendering should continue, a swapchain image is acquired, and then waited on using the
existing `Swapchain<Vulkan>` object to ensure the acquired image index has been read by the compositor.  This image
index is returned by the `engine.update()` function to the main hotham program, to be used during the rendering.

So far, so good.  But this does not tell us how the render context renders into the swapchain image.  If we examine
the render context, we see it has a `Swapchain` component.  A `Swapchain` object is created by the render context
in `new_from_swapchain_info`, which calls `Swapchain::new` with the `SwapchainInfo` structure created by calling
`from_openxr_swapchain`.  Looking at `rendering/swapchain.rs` we find this function on Android calls the function
`get_swapchain_images_with_ffr` to return the FFR and ordinary swap chain images.  For this it passes in the
`Swapchain<Vulkan>` handle it acquired from OpenXR.

The `get_swapchain_images_with_ffr` function returns the FFR and ordinary swapchain images directly from the OpenXR
API.  The swapchain information complete with images is returned to the `Swapchain::new` function which passes it
into the `create_framebuffers` function to create the framebuffer for the primary render pass.  Then the swapchain
information structure is dropped and the returned Swapchain object only contains the framebuffer with the swapchain
images in it.

`create_framebuffers` takes the swapchain images previously returned by OpenXR and simply adds the color and depth
images already created by the swapchain, and creates one framebuffer per swapchain image.  What is then done with
these swapchain images which are at this point only able to be accessed via this temporary image view?

Well, the framebuffer itself is simply a handle.  Vulkan provides no API to extract from the framebuffer the images
that went into it.  So we have two options:

The first is to use the same per frame framebuffer that the original rendering system would have otherwise used to
render to the screen.  If you do not need to add other attachments to your render pass that will be accessible by
the render pass, this is probably your best bet.  This removes any need for you to allocate images yourself into
which the scene will be rendered.

The render context framebuffer resolves the multi-sampled original colour attachment into the 3rd or 4th image in
the framebuffer, according to whether this is Android and FFR images are being used.  The zero based index of the
swapchain image within the framebuffer is held in the `RESOLVE_ATTACHMENT` constant, a `u32` within the render
context module itself.

There is nothing inherently wrong with two views onto the same image (including the swapchain images) existing,
as long as they are not written to at the same time.  Moreover, using the frame buffer does not preclude you from
using multiple render passes.  So there is some flexibility here.

The second option which will provide more control is to create your own frame buffer.  The function `from_openxr_swapchain` 
function which instantiates a SwapchainInfo structure will return the swapchain images and FFR images which can then be used
to construct an image view into the swapchain and FFR images.  This would then allow you to use multiple render passes which
include images not included in the primary render context framebuffers.

The swapchain module provides example code for creating an image view of the FFR images like so:

```rust,noplayground
    let ffr_image_view = vulkan_context
        .create_image_view(
            &swapchain_info.ffr_images[0].image,
            vk::Format::R8G8_UNORM,
            vk::ImageViewType::TYPE_2D_ARRAY,
            2,
            1,
            DEFAULT_COMPONENT_MAPPING,
        )
        .unwrap();
```

Note the two layers, for multi-views.  The format is `R8G8_UNORM`.  This means each sample is a 16 bit value.

If you use the method of resolving into the swapchain image that is used by the render context, make sure
that you update your attachments in the render pass creation to ensure all the images are of the correct
number of samples.  At present, this simply means passing the transient option to the `create_image` to
ensure it uses the provided constant number of samples (at present, four) for the MSAA images, and stipulate
the swapchain image as being single sample.


# Command ordering / pipeline barriers

You may wish to set up pipeline barriers to ensure that commands execute at the correct point in the execution
sequence. A common point of failure when learning how to use Vulkan is understanding the massively parallel nature
of GPU based rendering, where fragments or vertices can be processed many at once, not in a contiguous and
ordered sense but in an order which hopefully optimises the GPU's access to the memory it needs to read and write to.

As most people are CPU based programmers, the dominant paradigm of linear, synchronous processing is familiar and safe.
Moreover, even in the context of asychronous programming, or threading, some vestige of the linear nature of CPU bound
execution remains.  A highly non linear, multiprocess architecture such as a GPU uses many techniques for optimisation.
An example of this is foveated rendering, which is a tile based rendering paradigm in which greater levels of detail are
rendered for areas that are in direct focus for the eye.  Such tile based rendering solutions may store data in memory
and process it in a different way to traditional linear representations of an image as contiguous sequences of bytes in
memory. This has led to the concept in Vulkan for example of the image transition which occurs between stages or subpasses
which ensures the image is loaded in a way that is optimal for purpose.

This idea of a program which executes in parallel, out of order is critical to understanding dependencies.  A trivial example
would be the rendering of transparent objects.  Typically transparent objects in a scene are rendered last and in appropriate
order to ensure that blending happens in a deterministic way.  On a parallel architecture, this needs to be explicitly
requested, otherwise the transparent objects might render before the other objects in the frame buffer, or both before and after.

Pipeline barriers allow you to get more control over the ordering of your execution than the mere dependencies 
between passes.  Here is an example of a pipeline barrier using Ash.

```rust,noplayground
		let memory_barrier = [vk::MemoryBarrier::builder()
		    .src_access_mask(vk::AccessFlags::HOST_WRITE)
		    .dst_access_mask(vk::AccessFlags::MEMORY_READ)
		    .build()];
		unsafe { device.cmd_pipeline_barrier(command_buffer,
					    vk::PipelineStageFlags::BOTTOM_OF_PIPE,
					    vk::PipelineStageFlags::TOP_OF_PIPE,
					    vk::DependencyFlags::empty(),
					    &memory_barrier,
					    &[],
						&[]); };
```

The ubiquitous site GPU Open provides information about pipeline barriers [here](https://gpuopen.com/learn/vulkan-barriers-explained/).
There are other articles about them.  [Here is another](https://arm-software.github.io/vulkan_best_practice_for_mobile_developers/samples/performance/pipeline_barriers/pipeline_barriers_tutorial.html) that discusses best practices for mobile devices and specifically mentions the tiling example I touched on before.

The type of pipeline barrier described here can be used within a render pass to ensure specific commands execute *after* others in the pipe. 
This is outlined in the [Vulkan documentation](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/vkCmdPipelineBarrier.html) 
for the function which explains that *"If vkCmdPipelineBarrier was recorded inside a render pass instance, the first synchronization scope includes only commands that occur earlier in submission order within the same subpass."*  Similarly, it specifies that *"If vkCmdPipelineBarrier was recorded inside a render pass instance, the second synchronization scope includes only commands that occur later in submission order within the same subpass."*

A pipeline barrier can also occur outside a render pass, in which case **all** commands that occurred earlier, even in a prior subpass, fall into
the first synchronisation scope, while the second synchronisation scope becomes **all** subsequent commands including future subpasses.

What this means is that you can use render pass dependencies to control higher level flow between different subpasses, and use pipeline barriers
within this context to control execution flow within the subpasses themselves where relevant.  Combining these two techniques gives you full
control over the execution ordering.  The bottom of pipe/top of pipe example above constitutes a trivial example.

You should review the documentation [here](https://registry.khronos.org/vulkan/specs/1.3-extensions/man/html/VkPipelineStageFlagBits.html) for information on
the meaning of the different stages.  It is important to note that *VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT is equivalent to VK_PIPELINE_STAGE_ALL_COMMANDS_BIT with VkAccessFlags set to 0 when specified in the second synchronization scope, **but specifies no stage of execution when specified in the first scope.***  Ditto of course, for bottom of pipe and the second scope. 

# Culling

Here as a reminder is an example of using the culling compute shader.

```rust,noplayground
    let frame = &mut render_context.frames[render_context.frame_index];
    let cull_data = &mut frame.primitive_cull_data_buffer;
    cull_data.clear();
    
    for instanced_primitive in render_context.primitive_map.values() {
            let primitive = &instanced_primitive.primitive;
            for (instance, i) in instanced_primitive.instances.iter().zip(0u32..) {
        cull_data.push(&PrimitiveCullData {
                    bounding_sphere: instance.bounding_sphere,
                    index_instance: i,
                    primitive_id: primitive.index_buffer_offset | shaderlist.get(&primitive.index_buffer_offset).unwrap(),
                    visible: false,
        });
            }
    }

    // This is the VERY LATEST we can possibly update our views, as the compute shader will need them.
    render_context.update_scene_data(views, &gos_from_global, &gos_from_stage);
    
    // Execute the culling shader on the GPU.
    render_context.cull_objects(vulkan_context);

    // Begin the render pass, bind descriptor sets.
    render_context.begin_pbr_render_pass(vulkan_context, swapchain_image_index);	
```

The key function is `render_context.cull_objects`.  Examining what this function does, it sets the contents of the cull 
params storage buffer, into which the view projecion and number of objects in the buffer are placed, binds the compute 
command buffer which is separate from the existing command buffer being recorded, binds its descriptor sets, dispatches
the command to call the compute shader, and waits for it to complete. After execution, the `primitive_cull_data_buffer`
which is part of the render frame in question should have visibility updated for all objects.

The bounding sphere being pushed into the primitive cull data is what tells us whether an object is visible.  The 
multiplication of the center of the bounding sphere by the projection matrix in the clip planes data translates it into
Vulkan's normalized space.  Then, if the center of the sphere is closer to zero than the radius of the sphere, we know
that part of that bounding sphere rests on a point greater than zero that lies within the positive normalized space.

Also very important is `render_context.update_scene_data`  The `views` parameter passed to it originates from calling
`engine.xr_context.update_views()` which updates the eye views based on where OpenXR expects the users view will be at
the predicted display time.  `update_scene_data` takes these views and uses them to generate the frustum and camera
positions that will be included in the scene data buffer, which it updates.  `cull_objects` then makes use of this
updated scene data to populate its own cull parameters buffer.

If you have any non Mesh related primitive you need to calculate a bounding sphere for, `rendering/primitive.rs` exposes
a `pub fn calculate_bounding_sphere` which takes a slice of `Vec3` points and returns a `Vec4` with the w component equal
to the bounding sphere's radius.  There is also a related function for a given primitive, get_bounding_sphere_in_gos, 
which takes a parameter of an `&Affine3A` and transforms the primitive's bounding sphere into globally oriented space using 
that Affine.  These functions will be useful for those working with custom objects.
