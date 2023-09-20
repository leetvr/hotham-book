# Custom Rendering Part 2

One of the great difficulties of understanding Vulkan graphics pipelines is understanding how the different parts of the system work together. Let's start by listing the types of data you will access on the way to rendering your scene:

1. Push Constants.  These are defined during pipeline creation using the `push_constant_ranges` function of the `PipelineLayoutCreateInfo` builder pattern, and are pushed to the command buffer using the `device.cmd_push_constants` function prior to a command which uses the push constants using the data.  They are pushed on a per stage basis, typically once per draw command.
2. Vertex Data.  This is accessed by the vertex shader and the format of this data is defined when creating the pipeline in which you define the custom rendering behavior.  Specifically the `vertex_input_state` part of pipeline creation defines the layout and format of the data, which is then used in conjunction with the binding of vertex and index buffers to a command buffer prior to submitting draw commands relying on that format.
3. Objects located in the framebuffer. Note that the framebuffer exists on a per render pass basis.<br/><br/>In the code to the quadrics custom rendering example provided with Hotham 0.2.0, the custom rendering code upon switching to the Quadrics shader binds the adjusted descriptor sets but does not begin a new render pass.  In this case, this custom render code is piggy backing off the PBR shader's initial established render pass which bound the frame buffer containing the fixed foveated rendering attachments, depth buffer attachments, stencil attachments and so on.  Some objects within the frame buffer may be referenced within the shaders via descriptor sets bound to the command buffer.  For example, the GUI pipeline and set of render passes sets up a descriptor set to access the font texture which contains its unique visual information by defining it as an image sampler in its descriptor set at binding location zero, and then referencing this as a uniform sampler2D in the fragment shader code.
4. Objects not in the framebuffer but accessed via descriptor sets.  Each `hotham::rendering::buffer::Buffer` object can call a function `update_descriptor_set`, as can be seen in the code for `Frame::new` which updates the binding for each buffer.  It is in this function that a `vk::WriteDescriptorSet` is built with the `buffer_info` member set to the buffer, the binding number specified from the constants in `rendering/descriptors.rs`, and the set passed in from `Frame::new`.  This method is used to update both storage and uniform buffers.

Furthermore, there are other considerations as expressed previously:
- Does this change to the rendering require an entirely new render pass, or just a change to an existing shader?
- To produce the desired output, are multiple subpasses required? What are the dependencies between the subpasses including attachments later used as input attachments?
- What synchronisation methods are required to ensure commands execute in desired order?
- Can we use existing framebuffers or should we create our own?  How do we ensure the final output goes into the images acquired from the swap chain so they can be presented when `engine.finish` is called to the operating system by Open XR?

It is easy to overcomplicate the process of rendering in Vulkan. One of the reasons for this is that commands recorded into command buffers may execute out of order, and it can be challenging identifying the stage or region markers, memory barriers and so on that will enable the correct operation of the rendering pipeline. Another reason is the great amount of information that is needed in order to construct a pipeline, before a render pass involving that pipeline can even be instantiated.

As we can only focus on one area of concern at a time, let's focus first on how we should create a pipeline.

# Creating a pipeline

If this is your first time engaging in custom rendering, I recommend you leave the existing render pass alone and choose to render into an empty texture which you then display to screen. This prevents embarassing situations such as starting your example program and seeing garbage on the screen due to incorrect synchronisation or incorrect image format assumptions.  You can transition your code to render directly to the screen when you're confident the rendering is accurate.

The process of creating a pipeline is a long, involved one.  Let's summarize it:
1. You must create a pipeline layout.  This involves defining push constant ranges and descriptor set layouts which will be used during pipeline execution.
2. You must define the input assembly state.  This specifies the topology of the data received in the index buffers and/or vertex buffers.  Most of the time this will just be a triangle list.
3. You must define the vertex input state.  This is the definition of the input data to the vertex shader which include binding and attribute definitions.
4. You must define the viewport and scissors as appropriate. Most of the time, this is simply the whole screen.
5. You must define the rasterization state.  This includes how to draw the primitives assembled, how to cull polygons where appropriate, depth bias configuration if needed, and line width.
6. You must define the multi-sampling state, ie whether this render pass performs multi sample anti aliasing/down sampling.
7. You must define the depth stencil state, which includes whether to perform a depth test, whether to use a depth stencil test, and the minimum and maximum z.
8. You must define the color blend state, which defines for each relevant attachment its channels and how they are blended with existing fragments, if at all.
9. You must define a render pass with at least one subpass, including the render pass transitions, the subpass dependencies, and so on.
10. If you have defined attachments in your render pass, you must create a framebuffer to reference those attachments, after you have created the render pass.  This also implies the need to create images and image views to be referenced as color or depth attachments, etc.
11. You must load the compiled SPIR-V shader byte code and associate it with the relevant stages.
12. You must finally combine all of the above information into a `GraphicsPipelineCreateInfo` which is then provided to the `unsafe` function `create_graphics_pipelines` on the device instance in question.
13. You then need to store the pipeline, your descriptor sets, your framebuffers and render pass somewhere to use in the execution of your pipeline.

Lets now go through each of these stages and provide an example.  You can also follow along with the [Vulkan tutorial](https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Fixed_functions) for the fixed function stages to get more context.

# Pipeline Layout

```rust,noplayground
	let push_constant_range = vk::PushConstantRange::builder()
            .offset(0)
            .size(std::mem::size_of::<Material>() as _)
            .stage_flags(vk::ShaderStageFlags::FRAGMENT).build();

	let pipeline_layout_info = vk::PipelineLayoutCreateInfo::builder()
	    .push_constant_ranges(&[push_constant_range])
	    .set_layouts(&[render_context.descriptors.graphics_layout]).build();

	let pipeline_layout = unsafe { vulkan_context.device.create_pipeline_layout(&pipeline_layout_info, None).unwrap() };
```

In this example, we define for the fragment shader a single push constant range, stating that the push constant will be just the material being used to render this primitive. Secondly, we use the pre-defined graphics layout mentioned in the section on the Render Context to ensure the buffers for draw data, scene data, textures and so on are available when we use this pipeline.

# Vertex input state

```rust,noplayground
	// Vertex input state -- we choose the same position and vertex bindings and
	// corresponding attribute descriptions used by the default render context as
	// our shaders will mimic the PBR shader while allowing the alpha channel through
	// for final blending for seamless textures for all objects on the specific
	// render pipeline.
	let position_binding_description = vk::VertexInputBindingDescription::builder()
            .binding(0)
            .stride(std::mem::size_of::<Vec3>() as _)
            .input_rate(vk::VertexInputRate::VERTEX)
            .build();
	let vertex_binding_description = vk::VertexInputBindingDescription::builder()
            .binding(1)
            .stride(std::mem::size_of::<Vertex>() as _)
            .input_rate(vk::VertexInputRate::VERTEX)
            .build();
	let vertex_binding_descriptions = [position_binding_description, vertex_binding_description];
	let vertex_attribute_descriptions = Vertex::attribute_descriptions();
	
	let vertex_input_state = vk::PipelineVertexInputStateCreateInfo::builder()
            .vertex_attribute_descriptions(&vertex_attribute_descriptions)
            .vertex_binding_descriptions(&vertex_binding_descriptions);
```

Here we see the two types of data which are sent in per vertex by the typical PBR shader. Notice how stride and binding are used to define what is bound where. The `Vertex::attribute_descriptions()` function returns the attribute descriptions for both binding 0 and binding 1 defined above. If you examine the code, the `VERTEX_FORMAT` is just an alias for `R32G32B32_SFLOAT`, and the remaining attribute descriptions correspond with the locations defined in the vertex shader.

Another important thing to remember when constructing these structures is it is important to ensure the structures you pass in live long enough for the function to act on any references passed in.  For this reason, in the above code the vertex input descrioptions and the vertex binding descriptions are saved to a temporary variable which is then passed in to the `builder()` pattern to construct the final object. It is a good idea to do this as a design pattern to avoid accidentally causing a segmentation fault due to accessing unallocated memory.

# Input assembly state and viewport

```rust,noplayground
	// Input assembly state -- all objects passed to the input assembler are
	// a series of triangles composed of positions within the vertex buffer.
	let input_assembly_state = vk::PipelineInputAssemblyStateCreateInfo::builder()
            .topology(vk::PrimitiveTopology::TRIANGLE_LIST);

	let render_area = render_context.swapchain.render_area;
	    
	// Viewport State -- our pipeline affects the entire screen as designated by
	// the render area defined in the render context
	let viewport = vk::Viewport {
            x: 0.0,
            y: 0.0,
            width: render_area.extent.width as _,
            height: render_area.extent.height as _,
            min_depth: 0.0,
            max_depth: 1.0,
	};
	let viewports = [viewport];
	
	// Scissors
	let scissors = [render_area];

	// Viewport state: We select the entire viewport as the area to operate on
	let viewport_state = vk::PipelineViewportStateCreateInfo::builder()
            .viewports(&viewports)
            .scissors(&scissors);
```

Here, as described previously, the topology is set to triangle list.  The `render_context` contains a swapchain field which includes the render area which is typically the entire screen.  The maximum and minimum depth is also defined as the maximum extent possible.

Notice how the scissors is a simple `vk::Extent2D` and we use the same render_area to define both the viewport and the scissors, and follow the same design principle mentioned previously.

Also, for all of these patterns, although you may see example code within Hotham which does not perform the final `build()` call, you should do this to convert your object into the final format used later on.

# Rasterisation state

```rust,noplayground
	// Rasterization state -- we cull back faces, fill the polygons with their texture
	// and for now leave depth bias off, although we will test with depth bias enabled
	// at a later stage.
	let rasterization_state = vk::PipelineRasterizationStateCreateInfo::builder()
            .polygon_mode(vk::PolygonMode::FILL)
            .cull_mode(vk::CullModeFlags::BACK)
            .front_face(vk::FrontFace::COUNTER_CLOCKWISE)
            .rasterizer_discard_enable(false)
            .depth_clamp_enable(false)
            .depth_bias_enable(false)
            .depth_bias_constant_factor(0.0)
            .depth_bias_clamp(0.0)
            .depth_bias_slope_factor(0.0)
            .line_width(1.0);
	
	// Multisample state
	let multisample_state =
            vk::PipelineMultisampleStateCreateInfo::builder().rasterization_samples(vk::SampleCountFlags::TYPE_1);
```

Here we continue to the see the builder pattern without the final `build()` call.  This works in many instances due to the builder object containing in order the remaining fields that the final structure contains in order within its definition.

The builder pattern simply assigns to the fields of the struct in such a way as to ensure consistency, such as setting a count of attachments when attachments are specified and so on.  Thus taking a reference to the builder structure often results in the same outcome.  It is not recommended however to rely on this fortuitous behavior which may not always hold true.

Most of the time you will want to use FILL mode, however other options are available including outline only. Note how the determination of whether a face is front facing or backward facing is based on how the ordering of its vertices are specified. You should pay attention to this when you are constructing your meshes.

# Depth stencil state

```rust,noplayground
	let depth_stencil_state = vk::PipelineDepthStencilStateCreateInfo::builder()
            .depth_test_enable(true)
            .depth_write_enable(true)
            .depth_compare_op(vk::CompareOp::GREATER)
            .depth_bounds_test_enable(false)
            .min_depth_bounds(0.0)
            .max_depth_bounds(1.0)
            .stencil_test_enable(false);
```

Most of the time, unless you are doing something special, you will want to enable the depth test.  Despite what may be implied elsewhere, regardless of whether you use a depth stencil test or depth bounds test, if you enable the depth test you will need a depth attachment in your framebuffer with a suitable number of samples in order for a depth test to be performed, as the depth test needs somewhere to write its information to.  Pay attention to the fact that the `create_image` function within hotham 0.2.0 makes some assumptions about the number of samples you want in your image based on whether you specify usage flags as a transient attachment.  So if you are requesting a non MSAA depth buffer, you should specify non transient, or adapt the image creation code used in the vulkan context code for your own purposes.

In OpenXR, the Z value is more negative as it goes out from the camera.  In Vulkan, 1.0 is is the far view plane, while 0.0 is the near view plane. Hotham takes care of translating between this GOS of OpenXR and Vulkan's normalized space via the view projection matrices passed to the vertex shader.

The compare op is key. The GUI render pass, for example specifies that the comparison op is guaranteed to always evaluate to true via the comparison op ALWAYS.  This means it performs no depth test (because all panels are recorded at the same depth), but records the depth information for use in the next render pass. On the other hand, both the PBR render context / pipeline and the above example use the GREATER op.  This might seem counter intuitive given that most example code works off the description of samples being further away from the camera as depth increases, as described above.

The render code already performs a visibility test and knows that all objects drawn are visible between the right and left clip planes.  Typically, the LESS operation is used for opaque objects.  However, when transparent objects enter the scene, the ordering in which fragments are drawn matters. By rendering all GUI content to the screen at depth 1.0, the GUI fragments will always be on top, with their depth buffer information recorded, and then fragments with a greater depth value will be rendered next to allow color blending to occur.  This is known as transparency sorting.

# Color blending

```rust,noplayground
	let color_blend_attachment = vk::PipelineColorBlendAttachmentState::builder()
            .color_write_mask(
		vk::ColorComponentFlags::R
                    | vk::ColorComponentFlags::G
                    | vk::ColorComponentFlags::B
                    | vk::ColorComponentFlags::A,
            )
            .blend_enable(true)
            .src_color_blend_factor(vk::BlendFactor::SRC_ALPHA)
            .dst_color_blend_factor(vk::BlendFactor::ONE_MINUS_SRC_ALPHA)
            .build();

	let color_blend_attachments = [color_blend_attachment];
	
	let color_blend_state = vk::PipelineColorBlendStateCreateInfo::builder()
            .attachments(&color_blend_attachments);
```

Here we configure the color blending on a per attachment basis, specifically focusing on the color attachment.  Notice how each attachment is included within an array of attachments which is then passed to the builder function.  Note again the lack of a `build()` call; avoid this where possible.

Finally, to work with the comparison operator of GREATER specified earlier, this may not be the best blend factor.  Experiment to find one that works for you. Depending on which comparison operator is being used, you may find source blend factor one and destination blend factor one minus source alpha to be more appropriate.

# Creating your shaders

Let's discuss this example code:

```rust,noplayground
	// The following two lines are external to any function and are stored within
	// the module file, compiled to constant data on the heap.
	const SEMI_TRANSPARENT_VERT: &[u32] = include_glsl!("src\\shaders\\new.vert");
	const SEMI_TRANSPARENT_FRAG: &[u32] = include_glsl!("src\\shaders\\new.frag");
	// ... Additional code ...
	// SHADERS: Vertex and fragment only
        let (vertex_shader, vertex_stage) = create_shader(
            SEMI_TRANSPARENT_VERT,
            vk::ShaderStageFlags::VERTEX,
            vulkan_context,
        )
        .expect("Unable to create vertex shader");

        let (fragment_shader, fragment_stage) = create_shader(
            SEMI_TRANSPARENT_FRAG,
            vk::ShaderStageFlags::FRAGMENT,
            vulkan_context,
        )
            .expect("Unable to create fragment shader");
	
        let pipeline_shader_stages = [vertex_stage, fragment_stage];
```

When specifying the path to your shaders, usually they will be relative to the path to your crate / sub-project, where you have stored your `Cargo.toml` file.  While compiling your shaders, if there are any compilation errors, this will be included in the compilation output of your Rust file, and the Rust code will fail to compile if the shaders are unable to compile to SPIR-V format.

You do not need the shader modules once they have been used to create your pipeline and you can destroy them at the end of your pipeline creation to reclaim memory.

In this example, the pipeline stages returned from the creation of the shader modules will be passed to the pipeline creation function; hence they are included in the array `pipeline_shader_stages`.

# Creating your attachments

```rust,noplayground
	let rp_color_attachment = vk::AttachmentDescription::builder()
	    .format(COLOR_FORMAT)
	    .samples(vk::SampleCountFlags::TYPE_1)
	    .load_op(vk::AttachmentLoadOp::CLEAR)
	    .store_op(vk::AttachmentStoreOp::STORE)
	    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .initial_layout(vk::ImageLayout::UNDEFINED)
	    .final_layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL)
	    .build();

	let rp_depth_attachment = vk::AttachmentDescription::builder()
	    .format(DEPTH_FORMAT)
	    .samples(vk::SampleCountFlags::TYPE_1)
	    .load_op(vk::AttachmentLoadOp::CLEAR)
	    .store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .initial_layout(vk::ImageLayout::UNDEFINED)
	    .final_layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
	    .build();
```

To create a render pass, you need to define the attachments that will be used in it, and how they will be handled.  This is the purpose of the `AttachmentDescription` builder pattern. `COLOR_FORMAT` and `DEPTH_FORMAT` are defined in Hotham's lib.rs at the crate root.

You should make the number of samples in your attachments consistent with the multi sample state defined earlier.  If this number of samples is greater than one, you may need to specify resolve attachments to resolve into the final color image.

In this example we clear the image in accordance with the default clear colors specified by Hotham (which are specified when the framebuffer is bound).  We also clear the depth buffer to zeros.  This is in accordance with the transparency sorting outlined earlier.

In most cases you will not need to set the stencil load or store operations to anything other than don't care, unless you are going to be doing work with a depth stencil test. In those cases, you can refer to the online Vulkan documentation to determine an appropriate load/store op. In almost all cases, it is fine to leave the initial layout undefined and transition to the final optimal layout; but consult the documentation if you are unsure.

# Specifying your subpass and dependencies

```rust,noplayground
	let subpass = [vk::SubpassDescription::builder()
		       .pipeline_bind_point(vk::PipelineBindPoint::GRAPHICS)
		       .color_attachments(&[vk::AttachmentReference::builder()
				 .attachment(0)
				 .layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL)
					    .build()])
		       .depth_stencil_attachment(&vk::AttachmentReference::builder()
						   .attachment(1)
						   .layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
						   .build())
		       .build()];

 	let dependencies = [vk::SubpassDependency::builder()
	    .src_subpass(0)
	    .dst_subpass(vk::SUBPASS_EXTERNAL)
	    .src_access_mask(vk::AccessFlags::MEMORY_READ | vk::AccessFlags::MEMORY_WRITE)
	    .dst_access_mask(vk::AccessFlags::MEMORY_READ | vk::AccessFlags::MEMORY_WRITE)
	    .src_stage_mask(vk::PipelineStageFlags::ALL_GRAPHICS)
	    .dst_stage_mask(vk::PipelineStageFlags::ALL_GRAPHICS)
	    .build()];

```

Note how in this example code, each attachment has a number associated with it.  This corresponds to its position within the framebuffer. You also specify the format of the attachment again, as this information is being provided to a different part of the drivers to that which handles transitions.

In this example code, I have simply reproduced the dependencies specified for the GUI pipeline which were specified to ensure independence between the render pass and the next one in terms of memory accesses. Generally, the dependencies you set up are going to depend on how the pipeline/render pass will be interacting with other render passes to produce the final output.  I recommend a video linked on the Vulkan Youtube channel, [Vulkan render passes](https://youtu.be/yeKxsmlvvus).

# Creating the render pass

```rust,noplayground
	let attachment_list = [rp_color_attachment, rp_depth_attachment];
	let render_pass = unsafe {
	    vulkan_context.device.create_render_pass(&vk::RenderPassCreateInfo::builder()
						     .attachments(&attachment_list)
						     .subpasses(&subpass)
						     .dependencies(&dependencies).build(), None).expect("Unable to create render pass!")
	};
```

The `create_render_pass` is an unsafe function which returns a Result of the render pass to be unwrapped, or an error.  The render pass should be saved so that it can be used to begin render passes with the bound pipeline later on.

# Creating your framebuffer


```rust,noplayground
	let depth_image = vulkan_context.create_image(
                DEPTH_FORMAT,
                &render_context.swapchain.render_area.extent,
                vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT,
                1,
                1,
            )
            .unwrap();
			
	let attachment_views = [render_img.view, depth_image.view];
	
	let framebufferlist = unsafe { vulkan_context.device.create_framebuffer(&vk::FramebufferCreateInfo::builder()
										.render_pass(render_pass)
										.attachments(&attachment_views)
										.width(render_context.swapchain.render_area.extent.width)
										.height(render_context.swapchain.render_area.extent.height)
										.layers(1).build(), None)
				       .expect("Unable to create framebuffer!") };

```

Note that the default render pass for the PBR engine uses two layers.  Sometimes multiple layers may be required.  One example is for example with respect to multi views.  In this example render pass I have specified one layer only, and removed the transient attachment requirement in the call to `create_image` to ensure a single sample is returned.  If you need a multi-sample depth buffer, obviously you can specify the transient option to ensure Hotham picks four samples per pixel, if you need this option.

It is important when creating the framebuffer that the attachments list lasts long enough for the function to access it, otherwise a segmentation fault without further explanation in the program output may be encountered.

# Tying it all together - creating the pipeline

```rust,noplayground
	let create_info = vk::GraphicsPipelineCreateInfo::builder()
            .stages(&pipeline_shader_stages)
            .vertex_input_state(&vertex_input_state)
            .input_assembly_state(&input_assembly_state)
            .viewport_state(&viewport_state)
            .rasterization_state(&rasterization_state)
            .multisample_state(&multisample_state)
            .depth_stencil_state(&depth_stencil_state)
            .color_blend_state(&color_blend_state)
            .layout(pipeline_layout)
            .render_pass(render_pass)
            .subpass(0)
            .build();

	let create_infos = [create_info];

	println!("Creating pipeline");
	let pipelines = unsafe {
            vulkan_context.device.create_graphics_pipelines(
		vk::PipelineCache::null(),
		&create_infos,
		None,
            )
	};
	if pipelines.is_err() {
	    panic!("Unable to create custom rendering pipeline!");
	}

	let pipelines = pipelines.unwrap();
	    
	unsafe {
            vulkan_context
		.device
		.destroy_shader_module(vertex_shader, None);
            vulkan_context
		.device
		.destroy_shader_module(fragment_shader, None);
	}
```

Finally, all of the components described above are referenced to create the pipeline object.  Multiple pipelines can be created at once.  In this example, our pipeline is returned in `pipelines[0]`.  Notice we destroy the shader modules with a call to `destroy_shader_module` after the pipeline is created.

Then any structures you may wish to re-use in executing your pipeline should be saved.  For this reason I recommend wrapping your pipeline creation in a struct with a suitable impl block to create a new instance and store the pipeline, images and image views, frame buffer, the render pass object itself in the returned struct.

At this point you're halfway there!  Grab yourself a coffee, or maybe something stronger :/


To begin working with the existing PBR render context, it is important to know how it functions, so lets start by taking a look at the code of the `begin` function:

```rust,noplayground
pub unsafe fn begin(
    world: &mut World,
    vulkan_context: &VulkanContext,
    render_context: &mut RenderContext,
    views: &[xr::View],
    swapchain_image_index: usize,
) {
    // First, we need to walk through each entity that contains a mesh, collect its primitives
    // and create a list of instances, indexed by primitive ID.
    //
    // We use primitive.index_buffer_offset as our primitive ID as it is guaranteed to be unique between
    // primitives.
    let meshes = &render_context.resources.mesh_data;

    // Create transformations to globally oriented stage space
    let global_from_stage = stage::get_global_from_stage(world);

    // `gos_from_global` is just the inverse of `global_from_stage`'s translation - rotation is ignored.
    let gos_from_global =
        Affine3A::from_translation(global_from_stage.translation.into()).inverse();

    let gos_from_stage: Affine3A = gos_from_global * global_from_stage;

    for (_, (mesh, global_transform, skin)) in
        world.query_mut::<With<(&Mesh, &GlobalTransform, Option<&Skin>), &Visible>>()
    {
        let mesh = meshes.get(mesh.handle).unwrap();
        let skin_id = skin.map(|s| s.id).unwrap_or(NO_SKIN);
        for primitive in &mesh.primitives {
            let key = primitive.index_buffer_offset;

            // Create a transform from this primitive's local space into gos space.
            let gos_from_local = gos_from_global * global_transform.0;

            render_context
                .primitive_map
                .entry(key)
                .or_insert(InstancedPrimitive {
                    primitive: primitive.clone(),
                    instances: Default::default(),
                })
                .instances
                .push(Instance {
                    gos_from_local,
                    bounding_sphere: primitive.get_bounding_sphere_in_gos(&gos_from_local),
                    skin_id,
                });
        }
    }
```

Here, all visible meshes with a global transform are iterated and then their individual primitives are iterated.  A cloned copy of the primitive indexed on its position within the index buffer is inserted into the primitive map if it has not already been inserted, with an empty vec of instances.  Into the instances of that indexed primitive is inserted a single instance containing the bounding sphere of the primitive, its location and optional skin.

```rust,noplayground
    // Next organize this data into a layout that's easily consumed by the compute shader.
    // ORDER IS IMPORTANT HERE! The final buffer should look something like:
    //
    // primitive_a
    // primitive_a
    // primitive_c
    // primitive_b
    // primitive_b
    // primitive_e
    // primitive_e
    //
    // ..etc. The most important thing is that each instances are grouped by their primitive.
    let frame = &mut render_context.frames[render_context.frame_index];
    let cull_data = &mut frame.primitive_cull_data_buffer;
    cull_data.clear();

    for instanced_primitive in render_context.primitive_map.values() {
        let primitive = &instanced_primitive.primitive;
        for (instance, i) in instanced_primitive.instances.iter().zip(0u32..) {
            cull_data.push(&PrimitiveCullData {
                bounding_sphere: instance.bounding_sphere,
                index_instance: i,
                primitive_id: primitive.index_buffer_offset,
                visible: false,
            });
        }
    }
```

Here, the `cull_data` is a `Buffer<PrimitiveCullData>` which is a buffer of primitive instances created by the previous loop and indexed by their offset within the index buffer and the number of the instance. since one primitive may be drawn more than once at a different location.  Initially, each potentially visible primitive is set to invisible within the cull data buffer.

```rust,noplayground
    // This is the VERY LATEST we can possibly update our views, as the compute shader will need them.
    render_context.update_scene_data(views, &gos_from_global, &gos_from_stage);

    // Execute the culling shader on the GPU.
    render_context.cull_objects(vulkan_context);
```

Then, the culling shader is invoked with the compute shader's descriptor sets bound.  The culling shader only updates data within the primitive cull data buffer up to the number of draw calls recorded in the cull data parameters passed to the shader, which is set to the length of the primitive cull data buffer in the call to `cull_objects`.  Thus the intersection test with the clip plane is performed only for genuine instances.

After the render pass is begun, the primitives are drawn only if they're visible.  The following code from `draw_world` illustrates the process:

```rust,noplayground
    draw_data_buffer.clear();

    let mut instance_offset = 0;
    let mut current_primitive_id = u32::MAX;
    let mut instance_count = 0;
    let cull_data = frame.primitive_cull_data_buffer.as_slice();

	for cull_result in cull_data {
        // If we haven't yet set our primitive ID, set it now.
        if current_primitive_id == u32::MAX {
            current_primitive_id = cull_result.primitive_id;
        }

        // We're finished with this primitive. Record the command and increase our offset.
        if cull_result.primitive_id != current_primitive_id {
            // Don't record commands for primitives which have no instances, eg. have been culled.
            if instance_count > 0 {
                let primitive = &render_context
                    .primitive_map
                    .get(&current_primitive_id)
                    .unwrap()
                    .primitive;
                draw_primitive(
                    material_buffer,
                    render_context.pipeline_layout,
                    primitive,
                    device,
                    command_buffer,
                    instance_count,
                    instance_offset,
                );
            }

            current_primitive_id = cull_result.primitive_id;
            instance_offset += instance_count;
            instance_count = 0;
        }

        // If this primitive is visible, increase the instance count and record its draw data.
        if cull_result.visible {
            let instanced_primitive = render_context
                .primitive_map
                .get(&cull_result.primitive_id)
                .unwrap();
            let instance = &instanced_primitive.instances[cull_result.index_instance as usize];
            let draw_data = DrawData {
                gos_from_local: instance.gos_from_local.into(),
                local_from_gos: instance.gos_from_local.inverse().into(),
                skin_id: instance.skin_id,
            };
            draw_data_buffer.push(&draw_data);
            instance_count += 1;
        }
    }
```

Notice that in the first if block, the first iteration is skipped as the `cull_result.primitive_id` will be identical.  The code first checks whether the instance being examined is visible, and if it is, increments the instance count, after adding the draw data for the primitive (its GOS transformation and the inverse) into the draw data buffer.  The `draw_primitive` function extracts the material id from the primitive, pushes it as a push constant into the push constant data for the subsequent indexed command for the fragment shader stage, and then executes:

```rust,noplayground
    device.cmd_draw_indexed(
        command_buffer,
        primitive.indices_count,
        instance_count,
        primitive.index_buffer_offset,
        primitive.vertex_buffer_offset as _,
        instance_offset,
    );
```

You'll notice if you search for these `vertex_buffer_offset` and `index_buffer_offset` attributes within the `Primitive` struct that they are set within the `Primitive::new` function when a primitive is created.

Recall that when we created our rectangular mesh to draw text on in a previous section, we called Primitive::new using the vertex and index data in order to create the `Vec<Primitive>` that was provided to MeshData::new to create the data that eventually creates a Mesh.  `Mesh::new` allocates space for the meshdata passed to it and returns an `Id` from `id_arena`.

A final call is placed after the loop described above to draw the final visible primitive, if any, using the same methods described above.  The render pass is then ended.

What does this tell us about working with the existing render pass and its descriptor sets?  How can we adapt the same code to send multiple instanced primitives to different pipelines, while retaining the same structure for the accompanying primitive data?

First of all, we must look at the ownership of the data.  Each frame rendered holds a `primitive_cull_data_buffer` which is generated in the second section of code outlined above in `render_context.begin`, from the primitive_map previously populated at the start of the `begin()` function.  This `primitive_map` is cleared during `render_context.end()` and is *only used* in `systems/rendering.rs`.  Each primitive inserted into the primitive map is cloned and consists of simple u32 fields and vector data which reference the index and vertex buffers which were used in the creation of the primitive.

Consider that the only thing preventing multiple index and vertex buffers from being used here is the use of the index buffer offset as the key used to index into the hashmap `primitive_map`.  In the `custom_rendering.rs` of the custom rendering example within Hotham, this problem is worked around by oring the value of the u32 key with a suitable bit flag to indicate the separate shader.  The primitive cull data buffer maximum size is currently set at 100K.  131072 is the 17th power of 2, so that implies there is room to use this technique to handle potentially 16 unique bits of data within the key, which is plenty for any use that might be made for Hotham -- effectively it gives 65536 unique combinations of shader utilizing the one primitive map for culling.

Vulkan indexes into the index and vertex buffers currently bound by the render pass to draw a given instanced primitive.  This is not a problem if we bind the new index and vertex buffers along with the pipeline before the subsequent draw calls for the new set of primitives.  By binding each pipeline only once and then writing its related primitives in sorted order of pipeline, multiple pipelines can be executed within the same render pass.

It must be pointed out as well that there is no reason the same index and vertex buffers could not be used for multiple pipelines using different shaders.  The only thing which causes all primitives to be drawn via the same pipeline at present is the way the draw loop is constructed.  There is no reason you could not construct a struct to accept parameters for the construction of multiple pipelines which use the same layout and descriptor sets but different shaders, blend modes, and (if the index and vertex buffers were rebound), even different topologies or input data formats.

To prove this, let's create a custom rendering implementation that will make use of what we've seen so far.
