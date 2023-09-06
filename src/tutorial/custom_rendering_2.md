# Custom Rendering Part 2

One of the great difficulties of understanding Vulkan graphics pipelines is understanding how multiple pipelines work together.  There are several resources that your custom rendering could make access to:

1. Push Constants.  These are defined during pipeline creation using the `push_constant_ranges` function of the `PipelineLayoutCreateInfo` builder pattern, and are pushed to the command buffer using the `device.cmd_push_constants` function prior to a command which uses the push constants using the data.  They are pushed on a per stage basis.
2. Vertex Data.  This is accessed by the vertex shader and the format of this data is defined when creating the pipeline in which you define the custom rendering behavior.  Specifically the `vertex_input_state` part of pipeline creation defines the layout and format of the data, which is then used in conjunction with the binding of vertex and index buffers to a command buffer prior to submitting draw commands relying on that format.
3. Objects located in the framebuffer. Note that the framebuffer exists on a per render pass basis.  In the code to the quadrics custom rendering example provided with Hotham 0.2.0, the custom rendering code upon switching to the Quadrics shader binds the adjusted descriptor sets but does not begin a new render pass.  In this case, this custom render code is piggy backing off the PBR shader's initial established render pass which bound the frame buffer containing the fixed foveated rendering attachments, depth buffer attachments, stencil attachments and so on.  Some objects within the frame buffer may be referenced within the shaders via descriptor sets bound to the command buffer.  For example, the GUI pipeline and set of render passes sets up a descriptor set to access the font texture color attachment which contains its unique visual information by defining it as an image sampler in its descriptor set at binding location zero, and then referencing this as a uniform sampler2D in the fragment shader code.

Moreover, a graphics pipeline may do more than just produce output through the various shaders it defines.  A render pass, associated with a pipeline, may consist of multiple subpasses, the first of which writes to one attachment, the next of which reads from that previous sub-passes output.

It is easy to overcomplicate the process of rendering in Vulkan by getting lost in the terminology and making incorrect assumptions.  Do not make this mistake.

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
