# Loading Models

The simplest way to load your modules is demonstrated in the [simple-scene](https://github.com/leetvr/hotham/tree/main/examples/simple-scene) example provided within the hotham crate:

```rust,noplayground
    let mut glb_buffers: Vec<&[u8]> = vec![
        include_bytes!("../../../test_assets/floor.glb"),
        include_bytes!("../../../test_assets/left_hand.glb"),
        include_bytes!("../../../test_assets/right_hand.glb"),
    ];
    let models =
        asset_importer::load_models_from_glb(&glb_buffers, vulkan_context, render_context)?;
```

The `asset_importer::load_models_from_glb` function takes a `Vec<&[u8]>` as its first parameter.  Each `&[u8]` represents the binary content of a .glb file.  The `vulkan_context` and `render_context` are accessible within the engine object.

Including all models within the executable will quickly get out of hand if you're creating a professional app or game. Generally speaking, you will want to load
your models directly from an assets folder within your APK, or from
some writeable folder on the storage of the device.

To load from APK assets, Hotham provides a convenience function in `hotham::util` called `get_asset_from_path`.  Here is the method signature:

```rust,noplayground
pub(crate) fn get_asset_from_path(path: &str) -> Result<Vec<u8>>
```
Currently, this uses `ndk_glue::native_activity().asset_manager()` to access the asset manager and read the entire buffer into a `Vec<u8>`.

In my own example code which I used to learn the Hotham library, I abstracted the logic from this function into a separate library which cached the `native_activity` context rather than making multiple function calls for each file.  I then set up a list of glb files to read into a `Vec<Vec<u8>>` like this:

```rust,noplayground
    let mut vec_buff: Vec<Vec<u8>> = Vec::new();

    let asset_names= vec![ "glb/floor.glb", "glb/left_hand.glb",
			    "glb/right_hand.glb", "glb/damaged_helmet_squished.glb",
			    "glb/meditation retreat.glb", "glb/horned one.glb",
			    "glb/photo frame.glb",
    ];

    for asset in asset_names.into_iter() {
	vec_buff.push(match android_helper.read_asset_as_bytes(&asset) {
	    Ok(buff) => buff,
	    Err(_) => return Err(hotham::HothamError::Other(anyhow!("Can't open asset: {:?}", asset))),
	}.to_owned());
    };

    let glb_buffers: Vec<&[u8]> = vec_buff.iter().map(|x| &x[..]).collect();
```

In your Cargo.toml file, to do something like this, you'll want to reference an assets folder for `cargo apk` to pack into your APK like this:

```toml
[package.metadata.android]
assets = "D:\\hotham-main\\hotham-main\\examples\\simple-with-nav\\assets"
```

You could further simplify the above for loop to use `iter().map()` on the `Vec<&str>` to create a non-mutable `Vec<Vec<u8>>`, or use `as_slice()` to transform the returned `Vec<u8>`'s into slices before a single collect call.  I'll leave it up to you to experiment with what approach works best for you.

The native activity also exposes the `external_data_path` and `internal_data_path` properties, paths to the folders on the Android file system where public and private application data can be stored respectively.  These are Rust `Path` objects whose documentation is outlined [here](https://doc.rust-lang.org/nightly/std/path/struct.Path.html).

# Adding Models to the World

Once you have your models as above, you'll want to extract specific objects from the `Models`:

```rust,noplayground
pub type Models = HashMap<String, World>;
```

Models is really just a simple hashmap, with each object comprising a world of its own (queue the Seekers! 😂)

`add_model_to_world` takes this series of worlds and extracts one particular named set of objects for insertion into the world. It grabs the raw entities from the glb models that were loaded and inserts them with appropriate local or global transforms that reflect the transforms saved in the glb files.  It also handles meshes, skins, colliders (more on them later), parenting, and preservation of visibility.

The `add_model_to_world` function returns a `Result<Entity>`, which
can be unwrapped and used with `world.get<&mut LocalTransform>` or
`GlobalTransform` to adjust the transform of the entity with respect to its
parent or world space.

Here is an example of using it:

```rust,noplayground
    let negz = add_model_to_world("negz", models, world, None).unwrap();
    let posz = add_model_to_world("posz", models, world, None).unwrap();
    let rigid_posz = RigidBody {
	body_type: BodyType::Fixed,
	..Default::default()
    };

    let collider_posz = Collider::new(SharedShape::cuboid(2.5, 0.05, 2.5));
    
    world.insert_one(posz, rigid_posz);
    world.insert_one(posz, collider_posz);

    let rigid_negz = RigidBody {
	body_type: BodyType::Fixed,
	..Default::default()
    };

    let collider_negz = Collider::new(SharedShape::cuboid(2.5, 0.05, 2.5));

    world.insert_one(negz, rigid_negz);
    world.insert_one(negz, collider_negz);
 ```

This example requires a little explanation.  In my `glb/meditation retreat.glb` example file above, are six walls of a room, a rudimentary cube map.  posz and negz are actually the positive and negative y, as Blender switches the y and z axes.  Put simply, they are the ceiling and the floor respectively.  The final `None` parameter in `add_model_to_world` is the parent object.  If you specify it, it should be `Some(entity)`, where `entity` is the unwrapped result of adding the parent to the world.  Otherwise, `None` will make the object independent, which you will want if you want the physics engine to be able to control it.

Generally speaking, most of the time you will want your floor to be solid and immoveable.  That's where colliders and rigid bodies come in.  We'll discuss those in an upcoming tutorial.
