# Dynamic meshes and textures

So far, we have dealt with loading objects from glb files.  But what if
you wanted to load a PDF, or draw some fancy graphics and buttons on
a surface that were not known at compile time?  To do this, we need to
get a little more low level.

If you example the code for the loading of models from glb files, you'll
find that they call associated load functions within Hotham's variable
rendering component structs.  These are the structs of note:

```rust
rendering::{vertex::Vertex, primitive::Primitive, texture::Texture, material::{
    Material, MaterialFlags}, mesh_data::MeshData,
},
```

To break it down:
* A `components::mesh::Mesh` object is constructed from a `MeshData` object and a render context. 
* A `MeshData` object is constructed from a vector of `Primitive` structs.
* Each primitive is constructed out of a position buffer of `Vec3` structs, a vertex buffer, an index buffer of `u32`'s, a material (also a `u32`), and a render context.
* And the vertex buffer is constructed out of vertex objects.

We'll now break these down further to explain how to create your own meshes.

The vertex data, materials and textures as well as skins are all sent
into to vertex shader and fragment shader (located in the `src/shaders`
folder within the crate).
* So to map a texture onto a mesh, the shader needs to know the UV texture coordinates of each vertex.
* It also needs the normals of each vertex for other purposes such as lighting.
* Finally, if the object in question is skinned, it needs information about how the vertices are skinned.

A simple case is the creation of a mesh plane with two triangles.
Suppose we want to create a 4m x 4m plane onto which to project some
scene.  We would start by creating a position buffer containing the
locations of each vertex:

```rust
    let position_buffer: Vec<Vec3> = vec![
		[-2.0, 2.0, -2.0].into(),
		[-2.0, -2.0, -2.0].into(),
		[2.0, -2.0, -2.0].into(),
		[2.0, 2.0, -2.0].into()
    ];
```

This example lists the top left, bottom left, bottom right and top right
in order.  The index buffer is constructed of a series of triangles,
something like this:

```rust
    let index_buffer = [0, 1, 2, 0, 2, 3];
```

This re-uses vertices 0 and 2, the top left and bottom right, which are
shared between the two triangles of the square mesh plane.

Next, we need to specify the texture coordinates and vertex normals.
We can do this using Vertex::new, like so:

```rust
let vertex_buffer: Vec<Vertex> = vec![ 
    Vertex::new([0., 0., 1.].into(), [0., 0.].into(), 0, 0),
    Vertex::new([0., 0., 1.].into(), [0., 1.].into(), 0, 0),
    Vertex::new([0., 0., 1.].into(), [1., 1.].into(), 0, 0),
    Vertex::new([0., 0., 1.].into(), [1., 0.].into(), 0, 0),
];
```

The normal in this example is in the positive Z direction and is contained in the first parameter to `Vertex::new`.  Don't forget that this is *towards the camera* in OpenXR.  This example has the mesh plane in question situated forward 2.4 metres from the world origin, with the normal of light reflected off it bouncing back toward the viewer.

The texture coordinates are a `Vec2` of `f32` from `0.0` to `1.0`, with `0.0, 0.0` representing the top left of the texture and `1.0, 1.0` representing the bottom left.

Next we need a texture to map onto these coordinates.  The `render::texture::Texture` implementation provides a couple of functions, but for simplicity we'll only look at `Texture::empty`.  You need to pass a `vk::Extent2D` to specify the size of the empty texture that you’ll draw onto, like so:

```rust
    state.dyn_texture=Some(Texture::empty(
		&engine.vulkan_context,
		&mut engine.render_context, 
		vk::Extent2D::builder().width(1024).height(1024).build()
	));
```

This example is storing the new texture in a mutable state variable
as an `Option` for later unwrapping and adjustment.

Finally, we need a material to use this texture.  Other texturing
options like normal maps are available, most PBR maps are supported.
Consult the documentation or module code for more information.  In
this example we will simply create a material with a base color texture
and unlit workflow, such as you might use to display a 2 dimensional
image or rasterised text on screen.

```rust
let mut mat_flags = MaterialFlags::empty();
mat_flags.insert(MaterialFlags::HAS_BASE_COLOR_TEXTURE);
mat_flags.insert(MaterialFlags::UNLIT_WORKFLOW);

let dyn_material = Material {
    packed_flags_and_base_texture_id: mat_flags.bits() | (state.dyn_texture.as_ref().unwrap().index << 16),
    packed_base_color_factor: 0,
    packed_metallic_roughness_factor: 0,
};
```

Here we insert each material flag separately for clarity, although
these could simply be cast and or'ed together.  If you consult the code
for the Material struct implementation, as well as the shader common
code, you’ll see that the texture id is a word (16 bits) packed into the
most significant byte of the aptly named
`packed_flags_and_base_texture_id` field.  

In this case, we simply ignore the other fields.  If you want to create more complex textures with normal maps or other options, I would strongly suggest reading the material.rs in the rendering/ folder and the shader code to understand
how the various textures, materials and so on interact.

Next, you'll want to add this material to the materials buffer.  You'll
find something like this peppered throughout the hotham code:

```rust
let mat_dyntext = unsafe {
    engine.render_context.resources.materials_buffer.push(&dyn_material)
};
```

This is ensuring that the newly constructed material is available on
the GPU, and returns the material id.  You most likely won't need to
store this unless you want to duplicate the same texture on other
meshes.

Now that you have your beautiful empty texture and the coordinates
and objects to set up your plane, you can finish the job with something
like this:

```rust
let prim_dynscreen = vec![Primitive::new(&position_buffer[..], &vertex_buffer[..], &index_buffer[..], mat_dyntext, &mut engine.render_context)];

let meshdata_dynscreen = MeshData::new(prim_dynscreen);

let mesh_dynscreen = Mesh::new(meshdata_dynscreen, &mut engine.render_context);
```

We've called the object mesh_dynscreen as it represents a dynamic
"screen" onto which we’re going to draw text and other content.  This
later needs to get inserted into the world with other components to
position it properly.

```rust
world.spawn((Visible{}, mesh_dynscreen, LocalTransform {..Default::default()}, GlobalTransform::default()));
```

Because we've specified a fixed location for the plane in space in this
case, I've just used no translation or rotation on the Local and
GlobalTransform components above.  You might want to create your
plane or shape to align with an axis and then transform and translate it
or parent it to the camera or to your hands in the scene.

Finally, you'll want to draw something on your image before
uploading it to the GPU with something like:

```rust
engine.vulkan_context.upload_image(state.texture_buffer.as_ref().unwrap(), 1, vec![0], &(state.dyn_texture.as_ref().unwrap().image));
```

The texture buffer you are uploading should be in sRGBA format, with
each byte being `0..255` and the final byte being the alpha channel.
