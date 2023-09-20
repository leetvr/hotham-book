# The Render Context

The Render Context, located as `engine.render_context` with a struct type of `RenderContext`, will likely be the most useful to you in creating your own custom render pipeline, in conjunction with the previously mentioned `VulkanContext`.

It makes use of a number of struct types and constants defined elsewhere, particularly in the `hotham::rendering` hierarchy.

Here are some of the aspects of the render context which may be useful in building a pipeline.

# Descriptors

The `render_context` contains a field `descriptors` which is a struct containing the graphics_layout descriptor set and other descriptor sets using in the PBR pipeline.  Specifically, the `graphics_layout` binds the following types of data:

- **Draw Data** (for the vertex shader): This is the `gos_from_local` and `local_from_gos` transformations used to calculate the true position of an object from the vertex input to the shader.  The draw data also contains the skin id, as skinning affects the final position of a mesh.
- **Skins Binding** (for the vertex shader): As above, the skins has its own buffer of data relevant to calculating the position of meshes.  This is defined as part of the graphics binding.
- **Scene Data** (vertex and fragment stages): The Scene Data is a readonly uniform buffer containing the view projection matrices, camera positions, lights, and the params struct.  The params struct is solely for the purposes of debugging the shader.  The view projection matrices, on the other hand, are used to calculate the final position of each vertex given the position in globally oriented space.  There is one view projection for each eye: left first, right second.  The camera positions are similarly one for each eye, and are indicated in globally oriented space.
- **Texture Bindings** (fragment stage): This is pretty obvious: it binds the uniform sampler2D of textures, including all textures referenced by the materials buffer.  There are a maximum of 10000 of these.
- **Cube Texture Bindings** (fragment stage): These are the same but for cube textures.  These are used in calculating diffuse light for materials that do not have an unlit workflow (ie materials with metallic or roughness factor).

If you are creating a custom renderer which accesses textures loaded by the asset importer, or other functions within the rendering hierarchy, or if you need to access the GOS position of objects, or the view projection matrices, *you will most likely need to **bind this descriptor set**.*

The descriptor sets corresponding with this `graphics_layout` are located in the `sets` field of the descriptors struct.  There is one set allocated per eye.

The compute pipeline's descriptor set also present as `compute_layout` and `compute_sets`, with one descriptor set allocated per eye, to assist in making it easier to render the multi-view.  Typically, you will access the sets when binding your pipeline as:
```rust,noplayground
	slice::from_ref(&render_context.descriptors.sets[render_context.frame_index])
```

Or more fully:

```rust,noplayground
	vulkan_context.device.cmd_bind_descriptor_sets(
		command_buffer,
		vk::PipelineBindPoint::GRAPHICS,
		self.pipeline_layout,
		0,
		slice::from_ref(&render_context.descriptors.sets[render_context.frame_index]),
		&[],
	);
```

# Resources: The Index and Vertex Buffers

Assuming you are operating using a single set of vertex and index buffers, you'll want to bind these (or use your own, depending on the circumstances).  Index and vertex buffers are located under the resources field of the render context, along with a lot of other useful data used in rendering.  Here is an example of binding the index and vertex (and position) buffers:

```rust,noplayground
	vulkan_context.device.cmd_bind_vertex_buffers(
		command_buffer,
		0,
		&[render_context.resources.position_buffer.buffer, render_context.resources.vertex_buffer.buffer],
		&[0,0]);
	    
	vulkan_context.device.cmd_bind_index_buffer(
		command_buffer,
		render_context.resources.index_buffer.buffer,
		0,
		vk::IndexType::UINT32);

```

# Resources: Other Resources

Other resources located under the render context's `resources` field are:
- Materials buffer: This is a proper storage buffer which is pushed into whenever adding a new material
- Skins buffer: The buffer which is bound in the descriptor set mentioned above to give access to skinning data.
- Mesh Data Arena: An arena to store the mesh data 
- A texture sampler on repeat and a cube sampler.

All of these fields are named with obvious names such as `skins_buffer`, `mesh_data` etc.

# The Primitive Map

This is a `HashMap<u32, InstancedPrimitive>` named `primitive_map` which is cleared at the end of each render pass,
and is only populated from objects in the world during the beginning of the render pass and indexed on a suitable
key.  By default this key is the index buffer offset of the mesh indices of the primitive being referred to, however
this is a detail left up to the implementor.

It would for example be suitable to set a key containing a bit or multiple bits set to act as a shader flag.  In this
way a list of primitives sorted by this key will be emitted in order of their shader, allowing all the primitives in
one pipeline to be drawn before ending that render pass and switching to a separate render pass and pipeline for objects
that need to be handled specially.

Here is an example of the use of the primitive map:
```rust,noplayground
	let meshes = &render_context.resources.mesh_data;
	
	// Create transformations to globally oriented stage space
	let global_from_stage = stage::get_global_from_stage(world);
	
	// `gos_from_global` is just the inverse of `global_from_stage`'s translation - rotation is ignored.
	let gos_from_global =
            Affine3A::from_translation(global_from_stage.translation.into()).inverse();
	
	let gos_from_stage: Affine3A = gos_from_global * global_from_stage;

	let mut shaderlist=HashMap::new();

	for (_, (mesh, global_transform, skin)) in
            world.query_mut::<With<(&Mesh, &GlobalTransform, Option<&Skin>), &SemiTransparent>>()
	{
            let mesh = meshes.get(mesh.handle).unwrap();
            let skin_id = skin.map(|s| s.id).unwrap_or(NO_SKIN);
            for primitive in &mesh.primitives {
		let key = primitive.index_buffer_offset | SEMI_TRANSPARENT_BIT;
		
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
		shaderlist.insert(primitive.index_buffer_offset, SEMI_TRANSPARENT_BIT);
            }
	}
```

Essentially, each mesh with its accompanying components which contain a particular marker struct (in this case,
the `SemiTransparent` struct) are iterated and if it does not already exist in the primitive map with its corresponding
key (which in this case contains a bit set to identify the shader), a clone of the primitive is inserted with null instances,
and then its instances are populated with data from the primitive itself and its optional skin.  In this example, I also
populate a hashmap of shaders to reconstruct the primitive id to be added to the primitive cull data buffer.

This primitive map is later used to construct the data for sending to the culling shader, as well as to construct the
draw data for the visible primitives identified by the compute shader.

# Buffers and Frames

The `frames` field is an array of `Frame` structs.  Each `Frame` struct houses a number of buffers.

- The Command Buffer (`command_buffer`): The command buffer for recording commands.  Obviously there is one per frame,
so that fences on drawing one frame do not affect the next.
- The Draw Data Buffer (`draw_data_buffer`): This is pushed into for every indexed primitive drawn, to hold the data 
about the primitive's GOS position and skin id.
- The Primitive Cull Data Buffer (`primitive_cull_data_buffer`): This buffer is pushed into from the generated primitive
map mentioned above. It is used to construct a set of structs to be sent to the culling compute shader.  Objects are
typically pushed with their `visible` field set to `false`, and the compute shader sets to `true` those objects whose
bounding spheres fall within the left and right clip planes passed to the compute shader.
- The Cull Parameters Buffer (`cull_params_buffer`): A buffer of CullParams, used to pass uniform data to the compute
shader, not used for anything else.

Here is an example of using the cull data buffer:

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

You see here that the primitive cull data is populated with the primitive id which is the key which is used to look
up the object in the primitive map.  Because this key may change in format depending on your implementation, you should
make sure it is correctly set to get the data back about the visible primitives when you begin iterating the data for
your render psas.

The final call to `render_context.cull_objects` is executed after the scene data is updated.  This is important, as the
scene data is used to calculate the left and right clip planes, which will determine object visibility.  As the position
of the user's eyes may change from one millisecond to the next, we wait till the last minute to set this so that the true
scene visibility is as accurate as possible.

