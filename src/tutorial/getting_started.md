# Getting Started

The typical format of a Hotham program is going to include a `main.rs`,
a `lib.rs`, and additional modules for your separately written systems.  The
`main.rs` looks something like this:

```rust,noplayground
use hotham::HothamResult;

fn main() -> HothamResult<()> {
    simple_with_nav::real_main()
}
```

Within `lib.rs`, a configuration directive marks the true entry point of the program using `ndk_glue`:

```rust,noplayground
#[cfg_attr(target_os = "android", ndk_glue::main(backtrace = "on"))]
pub fn main() {
    println!("[HOTHAM_WITH_NAV] MAIN!");
    real_main().expect("Error running app!");
    println!("[HOTHAM_WITH_NAV] FINISHED! Goodbye!");
}
```

In the documentation of the now deprecated [ndk-macro](https://github.com/rust-mobile/ndk-glue/tree/main/ndk-macro), the full parameters to this attribute macro are described. You can configure the android logger, and override the default path to the crate.

In your `real_main()` function or within the marked `main()` itself, you start by initializing the `Engine`, and optionally checking for required OpenXR extensions.  You also create and initialize any state variable as well as any textures, models or meshes the program may need.  Then the tick loop occurs (discussed later in this section).

# Engine creation

To create the engine, the simplest way of doing it is:

```rust,noplayground
	let mut engine = Engine::new();
```

However, in some cases, such as if you want to enable some specific OpenXR extensions, you may wish to use `EngineBuilder`.  Here is a full example of using the `EngineBuilder` pattern with OpenXR, checking for supported extensions.

```rust,noplayground
    println!("Getting list of supported extensions...");
    let xr_entry = unsafe { xr::Entry::load().expect("Cannot instantiate OpenXR entry object!") };
    &xr_entry.initialize_android_loader();
    let xr_supported_extensions=&xr_entry.enumerate_extensions().unwrap();
    println!("Got list of supported extensions, eye gaze interaction is {}", xr_supported_extensions.ext_eye_gaze_interaction);

	let mut extension_set = xr::ExtensionSet::default().clone();
	if xr_supported_extensions.ext_eye_gaze_interaction {
	    extension_set.ext_eye_gaze_interaction = true;
	}
	
    let extensions_required = Some(extension_set);
    let application_name = Some("Hotham Navigation Test");
    let application_version = Some(1);
    let mut engine_builder = EngineBuilder::new();
    &engine_builder.application_name(application_name);
    &engine_builder.application_version(application_version);
    &engine_builder.openxr_extensions(extensions_required);
	let mut engine = engine_builder.build();	

```

There are two things of note here.  First, the code above assumes this is being built solely for an android device such as the Oculus Quest. Depending on your version of openxr, you may or may not need to include reference to initializing the Android loader as shown above.  Check the documentation for the version of openxr you are using in your dependencies for more information.  You may wish to adjust the openxr entry point loading code above to take into account your target os using `#cfg` directives.

Secondly, the engine_builder object has a lifetime which is based on the application name string.  The functions to set the engine builder fields in hotham v0.2.0 return &mut self, while build() takes ownership of self.  Therefore, to avoid borrow checker issues, using a pattern such as the above is suggested.

# State initialization

Concurrently with the engine being instantiated, you will want to initialize any models, meshes and textures you may need.

As an example, you may wish to initialize any action sets separate from the input context that you will use with OpenXR and store them in a place that will be accessible from the rest of your codebase:

```rust,noplayground
	state.eye_gaze_set = match engine.xr_context.instance.create_action_set("eye_gaze_interaction_scene_actions", "Eye Gaze Interaction Scene Actions", 1) {
	    Ok(actionset) => Some(actionset),
	    Err(_) => None,
	};

	if state.eye_gaze_set.is_some() {
	    println!("Eye gaze set created!  Creating eye gaze action...");
	    state.eye_gaze_action=match state.eye_gaze_set.as_ref().unwrap().create_action::<xr::Posef>("gaze_action", "Gaze Action", &[]) {
		Ok(gaze_action) => Some(gaze_action),
		Err(_) => None,
	    };

	    if state.eye_gaze_action.is_some() {
		println!("Eye gaze action was created!  Creating eye gaze path...");
		state.eye_gaze_path = match engine.xr_context.instance.string_to_path("/user/eyes_ext/input/gaze_ext/pose") {
		    Ok(some_path) => Some(some_path),
		    Err(_) => None,
		};

		let eyegaze_ip_path = engine.xr_context.instance.string_to_path("/interaction_profiles/ext/eye_gaze_interaction").unwrap();

		println!("Suggesting interaction profile bindings...");
		engine.xr_context.instance.suggest_interaction_profile_bindings(eyegaze_ip_path, &[
		    xr::Binding::new(state.eye_gaze_action.as_ref().unwrap(), state.eye_gaze_path.unwrap()),
		    ]);
		
		state.eye_gaze_space=match state.eye_gaze_action.as_ref().unwrap().create_space(engine.xr_context.session.clone(),
										 xr::Path::NULL,
										 xr::Posef::IDENTITY,
		) {
		    Ok(space) => Some(space),
		    Err(_) => None,
		};
									 
	    }
	}
```

In my example, the `ab_glyph::FontVec` of any font I may wish to use to present text to the user on screen is initialized:

```rust,noplayground
    state.android_helper = match AndroidHelper::new() {
		Ok(helper) => Some(helper),
		Err(_) => panic!("Could not get android context!"),
    };

	state.fontref = FontObject::new("fonts/font-001.otf",&mut state);
```

This is a typical pattern, in this example I pass in a state variable that has already stored a helper object to access APK resources.  Let's have a quick look at that:

```rust,noplayground
#[derive(Debug)]
pub struct FontObject {
    fontvec: Option<FontVec>,
    fontdata: Vec<u8>,
    fgcolor: FontColor,
}

impl Default for FontObject {
    fn default() -> Self {
		Self {
			fontdata: vec![],
			fontvec: None,
			fgcolor: FontColor::new(0.0, 0.0, 255.0, 255.0),
		} 
    }
}

impl FontObject {
    pub fn new(file_name: &str, state: &mut State) -> Self {
		let fontdata = state.android_helper.as_mut().unwrap().read_asset_as_bytes(file_name).unwrap();
		let fontvec = FontVec::try_from_vec(fontdata.clone()).unwrap();
    
		println!("Loaded {:?} glyphs from font-001", fontvec.codepoint_ids().count());
		Self {
			fontdata: fontdata,
			fontvec: Some(fontvec),
			..Default::default()
		}
    }

/// Further code follows...
}
```

`ab_glyph::FontVec` in this case does not implement Clone or Copy.  To ensure the object remains accessible to the main program, I store problematic objects within a `State` variable which can simply stay allocated and owned in the `real_main()` function and passed around as a borrowed mutable reference carefully. When being passed as a mutable reference only, the struct does not need to implement Clone, Copy or Debug, only Default (whether derived or specially written).

I recommend splitting the initialization up into separate functions, to avoid potential issues with the borrowing of the state variable.  For example:

```rust,noplayground
	init(&mut engine, &mut state);
    add_dynscreen(&mut engine, &mut state);
```

In my example here, the `init` function loads models and sets up physics engine properties.  Then the `add_dynscreen` function further manipulates the state variable to set up dynamic meshes and textures not loaded from a glb file.  The separation of concerns means that updates to the engine and the state that may cause borrow checker concerns can be minimised.

# The Tick Function

Before we look at loading models, a quick word about what happens post-initialization.

Typically a loop like this is used to run each of the game systems in sequence before updating the engine itself:

```rust,noplayground
    while let Ok(tick_data) = engine.update() {
        tick(tick_data, &mut engine, &mut state);
        engine.finish()?;
		if state.should_quit {
			break;
		}
    }
```

The tick_data contains the current and prior OpenXR session state,
which can be used to react to events such as a loss of focus or return to
focus of the app.

Basically, the update function handles polling and processing android
events, shutting down the app if a destroy event is received, handling
XR context state changes, and beginning the render frame.  The finish
function ends the render frame appropriately, and ends performance
timers for the current tick.

Here is an example of a tick function:

```rust,noplayground
fn tick(tick_data: TickData, engine: &mut Engine, _state: &mut State) {
    if tick_data.current_state == xr::SessionState::FOCUSED {
        hands_system(engine);
        grabbing_system(engine);
        physics_system(engine);
        animation_system(engine);
		let mover = Movement {
			world: &engine.world,
			stage_entity: engine.stage_entity,
			hmd_entity: engine.hmd_entity,
		};
		input::handle_input(&engine.input_context, &mover, _state);
		update_dynscreen(engine, _state);
        update_global_transform_system(engine);
        update_global_transform_with_parent_system(engine);
        skinning_system(engine);
        debug_system(engine);
    }

    rendering_system(engine, tick_data.swapchain_image_index);
}
```

In this example tick function, the rendering system is the last to be called, after the physics system, grabbing system, animation system and other systems have updated object positions and relevant textures, and global transforms have been updated by the transform systems listed. The rendering system is passed the image index in the swapchain that was calculated by `engine.update()` to ensure the correct frame buffer gets updated.

The following lines have been added for the sake of the dynamic texturing example which will be described in a later tutorial.  They are not hotham systems.

```rust,noplayground
		input::handle_input(&engine.input_context, &mover, _state);
		update_dynscreen(engine, _state);
```

# Debug System

If you wonder why your colors get all messed up when you press the buttons on the controllers, you probably have the debug system enabled, as per the second to last system showing in the example above.  This is a system designed to, as its comments indicate, help in debugging the fragment shader.  It makes some changes to the params variable which is a Vec4 passed to the shader.

Depending on the value of the third value in params, it will change the output color to display the base color, the normal, the occulusion, emission, roughness or metallic texture sampled.

If you don't need to debug your output textures, comment this line out or remove it to *turn this off*.  I will discuss the different systems that can be enabled shortly.

In the next section, we will look at things that typically happen in the initialisation of the program, including the loading of game models.
