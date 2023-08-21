# Tick Function & Systems

As described in the [Getting Started](getting_started.md) section, a hotham program updates the world each frame before rendering it within a tick function or tick loop, by calling a number of different systems which receive input and update the simulation.  Here again is an example of *some of* the functions that might be called during the tick function.

```rust,noplayground
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
        //debug_system(engine);
    }

    rendering_system(engine, tick_data.swapchain_image_index);
```

Each of these systems performs specific actions.  Some of them may not be exactly as you would like them. To make changes to these systems, you can use the code from one of these systems as a basis for your own version to run in its place.  Here is a description of what these various systems do:

1. **animation system**: For all <AnimationController> in the world, update the rotation, translation and scale of the target objects of this animation controller based on their currently set blend amounts.  The values which are animated through are based on the `blend_from` and `blend_to` values read in from the glb file the models are loaded from.
1. **audio system**: This queries any objects within the world with a `SoundEmitter` object, which is a wrapper over the `oddio` library.  It updates the position of audio based on the movement of the listener (ie deals with spatialisation), and updates the state of the audio emitters to continue playing their sound.  The position within the tick loop that you run this function is up to you; it is best to synchronise it with any other systems you run which may update the state of sound emitters within the world.
1. **debug system**: Reads input from the controllers and adjusts the value of the `scene_data.params` variable which is used by the shaders to determine the base output color displayed for PBR models.  This can be used to ensure that normal maps, occlusion, emission and so on have loaded correctly.  You can consult the `shaders/pbr.frag` code at the end of the main function to see what is displayed for different values of `params.z`.<br/><br/>Most likely, you want to turn this system off and capture the a, b, x, and y buttons yourself, and create your own action to update the scene data params that is more in line with your own program's input logic.
1. **draw gui system**: This system is within `systems::draw_gui`, and provides haptic feedback for any hover events, and paints the gui using the code in the `gui_context` implementation.  In the crab-saber example code, it is called *before* the rendering system, but after all other systems which may have updated the UI panels. One point of note is that in the crab-saber example, it runs *after* the haptics system, but performs haptic feedback.  You may wish to adjust the position of the haptics system in the loop to synchronise feedback from the GUI or other systems with the haptic system itself.
1. **hands system:** This updates the Global and Local transforms of the Left Hand and Right Hand models loaded into the scene, as well as updating the animation value on the animation controller to enable hand animations based on the value of each grip button.  This system should be run before the animation controller, which will update the models based on this adjusted blend value.  If you want to change the animation of the hands to use a different method of determining the blend value or final image, you can replace this call with your own custom system.<br/><br/>This system also updates the local and global transform of the currently grabbed entity from the grip.  If this is not a behavior you want, again you can adapt the code for this system to allow a more intuitive method of rotation and translation of the object.
1. **grabbing system**: This system will iterate which objects which have a `Grabbable` component are currently colliding with the left or right hand while the grip button is depressed, and change any rigid body to be kinematic position based.  This will also remove the `Parent` entity of the newly grabbed object, and set the grabbed_entity component of the hand in question to contain the newly grabbed object.  If the grip is released, it will transform any rigid body previously grabbed into one of type `Dynamic`, meaning the physics engine will then control its position.  It will then add a marker indicating the release of the object which will be removed on the next frame.<br/><br/>This may not be the behavior you want.  For example, you might want to restore the parent or the type of rigid body after the object has finished being manipulated.  If you want to do something like this, copy the system from the systems/grabbing.rs and modify the code to your taste. *You probably want to run the grabbing system **before** you run the physics system, to catch objects you want to grab before the physics system updates their position due to gravity/impulses.*
1. **haptic system**: This applies haptic feedback according to the feedback already applied this frame.  It is best to be run *after* you have executed any system which may apply haptic feedback.
1. **navigation system**: In the hotham_examples shared name space, an example navigation system for rotating, scaling and translating the entire scene is implemented to adjust the transform of the stage and subsequently the camera/head mounted display parented to it.<br/><br/>If you read any input to adjust the position of objects within your scene like this, it is usually best done after the animation system system, but before updating the global transforms or rendering the frame.  You can use the example code or write your own.  In the example above, I have replaced it with the call to `input::handle_input`, passing the input context, the state variable, and an android helper object which makes it easier to move objects about.  This is just an example of how you can write *whatever systems you want* and run them in *whatever logical order makes sense* for your application to function.
1. **pointer system**: This system handles collisions between items with a Pointer marker in their dynamic bundle, and the GUI panels in the scene.  It should be run *before* calling the system to draw the GUI.  You may wish to extend the ray casting logic included within this system to handle collisions of the pointers with other objects in the scene.
1. **physics system**: This updates the position of objects based on the physics simulation run by rapier3d.  It creates handles for any objects in the world which have a rigid body or collider but no handle in rapier, and updates the position of all rigid bodies and colliders from their world positions.  Then it runs the physics simulation, and updates the world based on the result of that simulation.  You'll usually want to run this *after* any input events have been handled (although this is design dependent), but *before* the rendering system or updating global transforms.  This should probably also run *before* the skinning system to ensure that any joints are in sync with updates to models made by the physics system.
1. **rendering system**: This renders the content that was updated as a result of running the previous systems.  This is best called near the end of the tick loop.
1. **skinning system**: This updates the joint matrices from the `GlobalTransform` associated with the skin in question.  It is best run *after* the global transforms have been updated using the `update_global_transform_system(engine)` and related systems.
1. **update global transform system**: This system, as the name suggests, updates the global transforms after previous systems have updated the corresponding `LocalTransform` objects. This simply updates all objects that have both a `LocalTransform` and `GlobalTransform` component.  To update the position of parented objects, the next system is used.
1. **update global transform with parent system**: This recursively iterates all objects with a parent, and then for the corresponding root/parent node, updates the global transform of the child with the parent projection.  This ensures that objects such as the head mounted display have a correct global transform which is relative to the stage object, and ensures any children whose parents are controlled by the physics engine fly in their respective groups.  It should be run *before the rendering system is called* to ensure objects appear in their updated positions.

# Recommendations

Pay careful attention to the ordering of the systems you run within your tick loop to ensure that the state of the virtual world remains sane.  Review the code of each one of the systems you are calling to ensure its actions are what you intend for your user interaction profile.  If they differ, clone the system logic and adapt it to your own needs.
