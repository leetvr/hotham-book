# Custom Rendering Part 4

## Tying it all together

Back at the beginning of the custom rendering section I mentioned transparency.  Transparency can be problematic to implement efficiently because of a few factors:
- The parallel processing nature of GPU based rendering
- The need to examine depth information when implementing transparency.

One method often used is what's known as a Z-only pass, where the first subpass updates the depth buffer without updating the color attachment, and the second subpass then renders into the color attachment with the depth attachment read only, first drawing all the opaque objects and then drawing the transparent/semi-transparent objects in order of distance from the camera, from farthest to nearest, thus allowing blending to be done according to depth.  This is also known as a Depth Pre-pass.

The problem with this approach is that when multiple transparent objects overlap, due to the transparent objects often not being written to the depth buffer, Vulkan cannot tell which transparent surface is on top.  Thus, if I render a red mesh plane with varying transparency from 0 to 255/1.0, behind which is a wall with a mural on it, the mural will show through, tinted by the red transparent surface. However, if I then render an object with fully transparent pixels on top of it, the fully opaque pixels will render fine while the overlapping transparent pixels will expose the pixels of the opaque object behind it, the wall with the mural.  Put simply, the translucent red screen will have a hole in it (regardless of how the blending of the alpha channels is handled).

The problem is outlined well [here](https://www.gamedev.net/blogs/entry/2271383-transparency-with-depth-sorting/) and is further discussed for OpenGL [here](https://www.khronos.org/opengl/wiki/Transparency_Sorting).

A Z-only pass *will slow down* your render.  But if you need real translucency the methods outlined above are your best bet.

However, the most common use for transparency is to have some pixels that are opaque with the remainder fully transparent.  An example of this is rendering text and other 2-dimensional graphics around a mesh plane but leaving the rest of the mesh plane transparent, so that the graphics can float in mid air.  This is used for things like heads up displays, billboards, and texture overlays.  If this is all you need, you're in luck.  Within the fragment shader, you have the opportunity to discard the fragment (pixel) if the alpha value is less than a threshold.

This is a case of "designing your solution to fit your use-case".  As the Gamedev article points out, *"Typically game developers get around that problem by simply designing their assets so that multiple transparent surfaces don’t stack up."*.  It isn't common in most games to see large numbers of floating transparent panels overlapping. More often you will have fully opaque surfaces with fully transparent pixels surrounding them to allow you to see through to the back of the object.  To do this in your fragment shader, you'll need to do something like:

```glsl
	if (outColor.a < 0.05) {
	    discard;
	}
```

If this was all you needed, we could end the tutorial here and say thanks for reading.

However, there are a couple of gotchas and further considerations for the rendering.

When converting your render pass to render to screen, there are a few places you can go wrong.  Let's start by having a look at an example of how to set up framebuffers.

```rust,noplayground
	let swapchain_info = SwapchainInfo::from_openxr_swapchain(&xr_context.swapchain, render_area.extent).unwrap();

	let mut framebuffers = vec![];

	let depth_image = vulkan_context.create_image(
            DEPTH_FORMAT,
            &render_context.swapchain.render_area.extent,
            vk::ImageUsageFlags::DEPTH_STENCIL_ATTACHMENT | vk::ImageUsageFlags::TRANSIENT_ATTACHMENT,
            2,
            1,
        ).unwrap();

	let msaa_image = vulkan_context.create_image(
            COLOR_FORMAT,
            &render_context.swapchain.render_area.extent,
            vk::ImageUsageFlags::COLOR_ATTACHMENT | vk::ImageUsageFlags::TRANSIENT_ATTACHMENT,
            2,
            1,
        ).unwrap();
	
	// Build an array of framebuffers as large as the number of swapchain images, with a separate depth buffer
	// and msaa image for each frame.
	for img_index in 0..swapchain_info.images.len() {
	    let swapchain_view = vulkan_context.create_image_view(&swapchain_info.images[img_index], COLOR_FORMAT,
								  vk::ImageViewType::TYPE_2D_ARRAY, 2, 1, DEFAULT_COMPONENT_MAPPING).unwrap();

	    let ffr_view = vulkan_context.create_image_view(&swapchain_info.ffr_images[0].image, vk::Format::R8G8_UNORM,
							  vk::ImageViewType::TYPE_2D_ARRAY, 2, 1, DEFAULT_COMPONENT_MAPPING).unwrap();
	
	    let attachment_views = [msaa_image.view, depth_image.view, ffr_view, swapchain_view];
	
	    let framebuffer0 = unsafe { vulkan_context.device.create_framebuffer(&vk::FramebufferCreateInfo::builder()
										.render_pass(render_pass)
										 .attachments(&attachment_views)
										 .width(render_context.swapchain.render_area.extent.width)
										 .height(render_context.swapchain.render_area.extent.height)
										 .layers(1).build(), None)
					.expect("Unable to create framebuffer!") };

	    framebuffers.push(framebuffer0);
	}
```

If you add this within your own code and run it, even with variable names adjusted, you'll run into a problem.  In Hotham 0.2.0,
`SwapchainInfo::from_openxr_swapchain` is `pub(crate)`, not pub. The simple solution is to alter the method signature to make it
pub, however you will also need to add a perfunctory comment with triple slashes to each of the methods you expose for documentation purposes.

Here are a few other places the conversion of your render pass can go wrong.

1. **Problem:** Objects display in the wrong places on the screen, and navigation is problematic<br/>
 **Solution:** Check the `instance_offset` parameter and ensure it is equivalent to the position the object's draw data has been inserted.
2. **Problem:** Scene displays normally but disappears after several seconds<br/>
**Solution:** Check to ensure you are clearing the draw_data buffer before populating it again.  This also applies to the cull data buffer
and primitive map.
3. **Problem:** Objects display out of order and in pieces, or a segmentation fault occurs<br/>
**Solution:** Check your render pass creation code to ensure the references that pass in attachment definitions live long enough.  A 
segmentation fault can occur due to short lived references passed to framebuffer creation for example. A segfault will also occur if
an end render pass is called without a matching begin render pass.
4. Make sure that the number of begin render passes is equal to the number of end render passes.
5. When drawing to dynamic textures, mind your casting.  Conversions between f32, u32 and usize when calculating buffer indexes can be
sensitive to bracketing/ordering and cause unexpected results.  Ditto for casting your buffer as a mut pointer; make sure the scoping
drops the mutable reference when you're done with it.

To make matters easier for my own future work, I created some basic classes to manage a collection of semi transparent mesh planes with
RefCell borrowing of a mutable reference to the buffer and graphics implementation functions on the mesh object itself.  Here's a short snippet.

```rust,noplayground
#[derive(Debug,Default)]
pub struct STMeshManager {
    mesh_list: HashMap<String, STMesh>,
}

impl STMeshManager {
    pub fn add(&mut self, engine: &mut Engine, mesh_name: String, transform: LocalTransform, width: f32, height: f32, ppi: f32, parent: Option<Parent> ) {
	let mesh=STMesh::new(engine, transform, width, height, ppi, parent);
	self.mesh_list.insert(mesh_name, mesh.clone());
    }

    pub fn get(&mut self,mesh_name: String) -> Option<&mut STMesh> {
	self.mesh_list.get_mut(&mesh_name)
    }

    pub fn len(&self) -> usize {
	self.mesh_list.len()
    }

    pub fn keys(&self) -> Vec<String> {
	self.mesh_list.keys().map(|s| s.clone()).collect()
    }
}
```

Here we are abstracting the process of adding a mesh to a list of meshes with a descriptive name.  The keys function clones the strings for safety to
ensure that once a key is added to the list, we don't mangle the data in the original reference.

```rust,noplayground
impl STMesh {
    pub fn new(engine: &mut Engine, transform: LocalTransform, width: f32, height: f32, ppi: f32, parent: Option<Parent> ) -> Self {
		let position_buffer: Vec<Vec3> = vec![[width*-0.5 , height*0.5, 0.].into(),
					      [width*-0.5, height*-0.5, 0.].into(),
					      [width*0.5, height*-0.5, 0.].into(),
					      [width*0.5, height*0.5, 0.].into()
	];

	let index_buffer = [0, 1, 2, 0, 2, 3];

	let vertex_buffer: Vec<Vertex> = vec![ Vertex::new([0., 0., 1.].into(), [0., 0.].into(), 0, 0),
					       Vertex::new([0., 0., 1.].into(), [0., 1.].into(), 0, 0),
					       Vertex::new([0., 0., 1.].into(), [1., 1.].into(), 0, 0),
					       Vertex::new([0., 0., 1.].into(), [1., 0.].into(), 0, 0),
	];

	// Since each object's size is specified in metres, the resolution of our texture
	// will be determined by the PPI that fits into the specified area, ie *100/2.54 = 39.37
	// conversion factor to inches or about 40 inches per metre by PPI.
	let x_width=width*39.37*ppi;
	let y_height=height*39.37*ppi;
	let dyn_texture=Texture::empty(&engine.vulkan_context, &mut engine.render_context,
				       vk::Extent2D::builder().width(x_width as u32).height(y_height as u32).build());
	let mut mat_flags = MaterialFlags::empty();
	mat_flags.insert(MaterialFlags::HAS_BASE_COLOR_TEXTURE);
	mat_flags.insert(MaterialFlags::UNLIT_WORKFLOW);
	
	let dyn_material=Material {
	    packed_flags_and_base_texture_id: mat_flags.bits() | (dyn_texture.index << 16),
	    packed_base_color_factor: 0,
	    packed_metallic_roughness_factor: 0,
	};
	
	let mat_dyntext = unsafe { engine.render_context.resources.materials_buffer.push(&dyn_material) };
	let prim_dynscreen = vec![Primitive::new(&position_buffer[..], &vertex_buffer[..], &index_buffer[..], mat_dyntext, &mut engine.render_context)];
	let meshdata_dynscreen = MeshData::new(prim_dynscreen);
	let mesh_dynscreen = Mesh::new(meshdata_dynscreen, &mut engine.render_context);
	let mut entity;

	if parent.is_some() {
	    entity=engine.world.spawn((renderimagebuff::SemiTransparent {}, mesh_dynscreen,
				parent.unwrap(), transform.clone(), GlobalTransform::default()));
	} else {
	    entity=engine.world.spawn((renderimagebuff::SemiTransparent {}, mesh_dynscreen,
				transform.clone(), GlobalTransform::default()));
	}
	let texture_buffer = vec![0; x_width.round() as usize*y_height.round() as usize*4];
	let t_width=x_width.round() as u32;
	let t_height=y_height.round() as u32;

	Self {
	    entity: entity,
	    texture_buffer: RefCell::new(texture_buffer),
	    material: Rc::new(dyn_material),
	    texture_object: Rc::new(dyn_texture),
	    t_width: t_width,
	    t_height: t_height,
	}
    }

    pub fn update(&self, vulkan_context: &VulkanContext) {
	// Caller must take care to drop any exclusive/&mut references so that borrow will succeed.
	vulkan_context.upload_image(&self.texture_buffer.borrow(), 1, vec![0], &self.texture_object.image);
    }
```

This takes our dynamic texturing code and wraps it into a function to return an entity added to the
world passed in, with a `RefCell` for the singular object we will need to mutate, and material and
texture objects stored as immutable `Rc`'s. 

The update function uploads the changed texture to the GPU.

Further content can be added to perform basic 2D graphics functionality, or it can be farmed out
to 3rd party modules:

```rust,noplayvground
	pub fn line(&mut self, x1: u32, y1: u32, x2: u32, y2: u32, color: font_texturing::FontColor) {
		let mut buff = self.texture_buffer.borrow_mut();
		let buff = buff.as_mut_slice();
		let (mut sx, mut sy)=(x1 as f32, y1 as f32);
		let (ex, ey)=(x2 as f32, y2 as f32);
		let mut iters=0;
		if (ex-sx).abs() > (ey-sy).abs() {
			iters=(ex-sx).abs() as u32;
		} else {
			iters=(ey-sy).abs() as u32;
		}

        if iters == 0 {
			iters=1;
		}
	
		let dx=(ex-sx)/(iters as f32);
		let dy=(ey-sy)/(iters as f32);
		for i in 0..iters {
			let buff_pos=((sx as u32)+(sy as u32)*self.t_width)*4;
			let buff_slice=&mut buff[buff_pos as usize..(buff_pos+4) as usize];
			buff_slice.copy_from_slice(&[color.r as u8, color.g as u8, color.b as u8, color.a as u8]);
			sx+= dx; sy +=dy;
		}
    }
}
```

You're going to have a lot of use statements in your custom renderer code.  Here's the ones in my custom
rendering module:

```rust,noplayground
use std::{slice, collections::HashMap, borrow::BorrowMut};
use hotham::{rendering::{vertex::Vertex, primitive::Primitive, texture::{Texture, DEFAULT_COMPONENT_MAPPING}, material::{Material, MaterialFlags},
			mesh_data::MeshData, buffer::Buffer, resources::Resources, image::Image, swapchain::SwapchainInfo}, COLOR_FORMAT, DEPTH_FORMAT, contexts::XrContext, VIEW_COUNT, vk::Handle};
use hotham::contexts::{vulkan_context::VulkanContext, render_context::RenderContext};
use hotham::contexts::render_context::{Instance, InstancedPrimitive};
use hotham::rendering::resources::{DrawData, PrimitiveCullData};
use hotham::ash::vk;
use glam::{Vec3, Mat4, Affine3A};
use hotham::vk_shader_macros::include_glsl;
use hotham::contexts::render_context::create_shader;
use hotham::components::stage;
use hotham::components::skin::{Skin, NO_SKIN};
use hotham::components::{GlobalTransform, Mesh, Visible};
use hotham::Engine;
use hecs::{With, World};
use hotham::xr;
use hotham::systems::rendering::draw_primitive;
```

Here are the fields I ended up having to store in order to implement my transparent rendering:

```rust,noplayground
pub struct CustomRenderBuff {
    pub pipeline: vk::Pipeline,
    pub pipeline_layout: vk:: PipelineLayout,
    pub framebuffers: Vec<vk::Framebuffer>,
    pub swapchain_info: SwapchainInfo,
    pub renderpass: vk::RenderPass,
    prim_index: u32,
}
```

This list could probably be reduced.  I originally stored the u32 of the primitive into which I would be
rendering the scene output.  That way I could filter that primitive out of the world while drawing it.

The render pass is needed to begin the render pass.  The pipeline needs to be bound to the render pass.  
The pipeline layout is needed for when your descriptor sets are bound.

The framebuffers, if you are doing full custom rendering to replace the default PBR pass, should be
a vector or list of framebuffers, one for each swapchain image index.  Each framebuffer can re-use
the same color and depth attachments etc, as the contents of the attachments are no longer needed
once the swapchain image is rendered and can be discarded.

Other than that, I stored the swapchain information just in case I need to access those again later
as iterating the swapchain through openxr is an expensive operation.

For the implementation of this custom render context I decided to have the rendering system function
form part of the functions that can be called on mut self.

```rust,noplayground
    fn rendering_system_inner(&self, world: &mut World, vulkan_context: &VulkanContext,
			      render_context: &mut RenderContext, views: &[xr::View],
			      swapchain_image_index: usize) {
	    unsafe {
            self.begin(
				world,
				vulkan_context,
				render_context,
				views,
				swapchain_image_index,
            );
            self.render_pass2(vulkan_context, render_context, swapchain_image_index);
            self.end(vulkan_context, render_context);
		}
    }
    
    pub fn rendering_system(&self, engine: &mut Engine, swapchain_image_index: usize) {
		let world = &mut engine.world;
		let vulkan_context = &mut engine.vulkan_context;
		let render_context = &mut engine.render_context;
	
		// Update views just before rendering.
		let views = engine.xr_context.update_views();
		self.rendering_system_inner(world, vulkan_context, render_context, views, swapchain_image_index);	    
    }
```

We've already covered a lot of the `CustomRenderBuff::new` function, but here is part of the pattern that wasn't
covered in the pipeline setup in part 1

```rust,noplayground
	let ffr_reference = vk::AttachmentReference::builder()
	    .attachment(2)
	    .layout(vk::ImageLayout::FRAGMENT_DENSITY_MAP_OPTIMAL_EXT)
	    .build();
	
	let mut ffr_info = vk::RenderPassFragmentDensityMapCreateInfoEXT::builder()
            .fragment_density_map_attachment(ffr_reference).build();
	let attachment_list = [rp_color_attachment, rp_depth_attachment, rp_ffr_attachment, rp_resolve_attachment];
	let view_masks = [!(!0 << VIEW_COUNT)];
	let mut multiview = vk::RenderPassMultiviewCreateInfo::builder()
            .view_masks(&view_masks)
            .correlation_masks(&view_masks)
            .build();

	let render_pass = unsafe {
	    let mut rp_info=vk::RenderPassCreateInfo::builder()
		.attachments(&attachment_list)
		.subpasses(&subpass)
		.dependencies(&dependencies)
		.push_next(&mut multiview);
    	    let rp_info=rp_info.push_next(&mut ffr_info).build();

	    vulkan_context.device.create_render_pass(&rp_info, None).expect("Unable to create render pass!")
	};
```

FFR images are added via the fragment density map extension and do not form part of the `SubpassDescription` builder
pattern.  For those looking to replicate the type of setup used in the PBR render pass, here are the attachments:

```rust,noplayground
	let rp_color_attachment = vk::AttachmentDescription::builder()
	    .format(COLOR_FORMAT)
	    .samples(vk::SampleCountFlags::TYPE_4)
	    .load_op(vk::AttachmentLoadOp::CLEAR)
	    .store_op(vk::AttachmentStoreOp::STORE)
	    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .initial_layout(vk::ImageLayout::UNDEFINED)
	    .final_layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL)
	    .build();

	let rp_depth_attachment = vk::AttachmentDescription::builder()
	    .format(DEPTH_FORMAT)
	    .samples(vk::SampleCountFlags::TYPE_4)
	    .load_op(vk::AttachmentLoadOp::CLEAR)
	    .store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .initial_layout(vk::ImageLayout::UNDEFINED)
	    .final_layout(vk::ImageLayout::DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
	    .build();

	let rp_ffr_attachment = vk::AttachmentDescription::builder()
	    .format(vk::Format::R8G8_UNORM)
	    .samples(vk::SampleCountFlags::TYPE_1)
	    .load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .initial_layout(vk::ImageLayout::UNDEFINED)
	    .final_layout(vk::ImageLayout::FRAGMENT_DENSITY_MAP_OPTIMAL_EXT)
	    .build();


	let rp_resolve_attachment = vk::AttachmentDescription::builder()
	    .format(COLOR_FORMAT)
	    .samples(vk::SampleCountFlags::TYPE_1)
	    .load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .store_op(vk::AttachmentStoreOp::STORE)
	    .stencil_load_op(vk::AttachmentLoadOp::DONT_CARE)
	    .stencil_store_op(vk::AttachmentStoreOp::DONT_CARE)
	    .initial_layout(vk::ImageLayout::UNDEFINED)
	    .final_layout(vk::ImageLayout::COLOR_ATTACHMENT_OPTIMAL)
	    .build();
```

In this example, the resolve attachment and the FFR attachments are provided to us by Vulkan
through openxr and the swapchain.  The code at the top of part 4 outlines the setup process.

Here is how the render pass is set up:

```rust,noplayground
    fn start_pass(&self, vulkan_context: &VulkanContext, render_context: &mut RenderContext, swapchain_image_index: usize) {
		let fb = self.framebuffers[swapchain_image_index];
		let rp_begin_info = vk::RenderPassBeginInfo::builder()
			.framebuffer(fb)
			.render_pass(self.renderpass)
			.render_area(render_context.render_area())
			.clear_values(&hotham::contexts::render_context::CLEAR_VALUES).build();
	
		let command_buffer = render_context.frames[render_context.frame_index].command_buffer;

	
		unsafe {
			println!("Beginning render pass using framebuffer {:?}, image index {}", fb, swapchain_image_index);
			vulkan_context.device.cmd_begin_render_pass(command_buffer, &rp_begin_info, vk::SubpassContents::INLINE);
	
            vulkan_context.device.cmd_bind_pipeline(
				command_buffer,
				vk::PipelineBindPoint::GRAPHICS,
				self.pipeline,
            );

            vulkan_context.device.cmd_bind_descriptor_sets(
				command_buffer,
				vk::PipelineBindPoint::GRAPHICS,
				self.pipeline_layout,
				0,
				slice::from_ref(&render_context.descriptors.sets[render_context.frame_index]),
				&[],
            );
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
		};
    }
```

This is pretty simple.  Bind the framebuffers, pipeline, vertex buffers, index buffers and descriptor sets.

```rust,noplayground
    fn end(&self, vulkan_context: &VulkanContext, render_context: &mut RenderContext) {
		// OK. We're all done!
		render_context.primitive_map.clear();
		render_context.end_pbr_render_pass(vulkan_context);
    }
```

End is trivial.  Close off your render pass and clear any temporary buffers.

Here is the real meat and potatoes of the rendering code that works with the populated cull data buffer as
described earlier:

```rust,noplayground
    unsafe fn render_pass2(&self, vulkan_context: &VulkanContext, render_context: &mut RenderContext, swapchain_image_index: usize) {
		// Parse through the cull buffer and record commands. This is a bit complex.
		println!("Entering RP2");
		let device = &vulkan_context.device;
		let frame = &mut render_context.frames[render_context.frame_index];
		let command_buffer = frame.command_buffer;
		let draw_data_buffer = &mut frame.draw_data_buffer;
		let material_buffer = &mut render_context.resources.materials_buffer;
		let mut instance_offset: u32 = 0;
		let mut current_primitive_id = u32::MAX;
		let mut current_shader = 0;
		let instance_count = 1;
		let cull_data = frame.primitive_cull_data_buffer.as_slice();
		draw_data_buffer.clear();
	
		let fov = render_context.cameras[0].position_in_gos();
		let camera_pos = Vec3::new(fov.x, fov.y, fov.z);
		let mut visible_solids: Vec<PrimitiveCullData> = cull_data.iter()
			.filter(|cull_res| cull_res.visible && cull_res.primitive_id != self.prim_index)
			.map(|p| p.clone()).collect();
			
		visible_solids.sort_by(|a,b| {
			if a.primitive_id != b.primitive_id {
				a.primitive_id.cmp(&b.primitive_id)
			} else {
				let dist1=Vec3::new(a.bounding_sphere.x, a.bounding_sphere.y, a.bounding_sphere.z);
				let dist2=Vec3::new(b.bounding_sphere.x, b.bounding_sphere.y, b.bounding_sphere.z);
				dist1.distance_squared(camera_pos).partial_cmp(&dist2.distance_squared(camera_pos)).unwrap()
			}
		});
		
		for prim in visible_solids {
			let new_shader = prim.primitive_id & SEMI_TRANSPARENT_BIT;
			
			if new_shader > 0 {
				unsafe { device.cmd_pipeline_barrier(command_buffer,vk::PipelineStageFlags::BOTTOM_OF_PIPE,
     	           			vk::PipelineStageFlags::BOTTOM_OF_PIPE, vk::DependencyFlags::empty(),
							&[], &[], &[]); };
				current_shader = new_shader;
			}
	    
			let instanced_primitive = render_context
                .primitive_map
                .get(&prim.primitive_id)
                .unwrap();
				
			let instance = &instanced_primitive.instances[prim.index_instance as usize];
			let draw_data = DrawData {
				gos_from_local: instance.gos_from_local.into(),
				local_from_gos: instance.gos_from_local.inverse().into(),
				skin_id: instance.skin_id,
			};
		
			instance_offset = draw_data_buffer.push(&draw_data);
			
			draw_primitive(
				material_buffer,
				self.pipeline_layout,
				&instanced_primitive.primitive,
				device,
				command_buffer,
				instance_count,
				instance_offset,
			);
		}
    }
```

This code is a bit messy.  The instance offset updating from the original custom render example is no longer required, 
nor is the `current_shader` or the `current_primitive_id` or the instance count which will always be one. 
The custom render code for drawing quadrics manually updated `instance_offset` as it parsed through which primitives
were visible and pushed their data into the draw data buffer.  However, the vertex shader indexes into the draw data 
buffer using the gl instance index generated by `instance_offset`.  Thus it made sense to simply use the index returned by the buffer push.

The doubled up draw loop visible in the custom rendering example and the PBR render code within `render_context.rs`
has been reduced to a filter/map/collect statement.  The sort statement sorts primitives into order of the instance
and shader bit, and then into order of distance from the camera, in this case using the left eye.

The pipeline barrier inserts a conservative barrier to ensure all prior commands are finished before rendering the
semi transparent objects.

For brevity, I will not reproduce the same example code for the population of the cull data buffer here.

We have now covered:
- Pipeline creation
- Accessing information about the swapchain
- Creating framebuffers
- Mimicking the attachment setup of the primary PBR render pass.
- Culling non visible objects
- Beginning the render pass and binding the framebuffer, descriptors and other important buffers
- Iterating the visible objects to execute draw commands
- Adding pipeline barrier commands

The example code produced here is an inefficient, first principles attempt at combining Hotham's IBL/PBR model
with partial and full transparency using a combination of different techniques.  If I was to design this for
efficiency, I would probably:
- Split the transparent/semi-transparent objects and opaque objects into separate sub-passes.
- Split the two categories of object out into separate lists and only sort the transparent objects
- Allow the opaque objects pass to populate the depth buffer
- Insert a pipeline barrier to ensure depth buffer writes had finished
- Draw the remaining objects in order of distance from the camera

I'd also consider whether I truly need partial transparency, since if you don't need that, the use of the
discard keyword makes full transparency trivial.
