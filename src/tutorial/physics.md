# Physics

Hotham uses rapier3d as its physics engine, with a thin wrapper over
the top of RigidBody to allow it to run the physics simulation on any
physics engine controlled objects.

To adjust physics within the world, you can adjust the
`engine.physics_context` fields such as the vector for gravity.  By default,
no gravity exists and any newly created objects will have no impulse on
them unless this is set up.

Example:
```rust,noplayground
    engine.physics_context.gravity = [0., -9.81 / 2., 0.].into();
```

To set the gravity to half earth normal.

- Upon releasing a grabbed object, an impulse from the hand may be imparted to the object.
- This means that without setting up gravity, released rigid bodies
    which have their body type set back to dynamic will spin or move
    in space according to the impulse that was on them at the time of
    release.  This can cause them to fly off into space.

To make a game object a rigid body, simply insert a hotham
RigidBody struct into the dynamic bundle for the associated entity.
Rigid bodies receive their initial position from their parent object/entity,
and then update the associated game objects as the physics pipeline
unfolds progressively across engine ticks. 

Fixed rigid bodies do not move and are considered to have infinite
mass.

For more information, visit the documentation for rapier3d at [https://www.rapier.rs/](https://www.rapier.rs/).

# Colliders

In hotham, there are several ways of specifying colliders.
- You can specify them as part of the glb files you load, by creating an object name suffixed by one of the following tags:
	- **.HOTHAM_COLLIDER_WALL**: A wall colliders
	- **.HOTHAM_COLLIDER_SENSOR**: A sensor only collider
	- **.HOTHAM_COLLIDER**: An ordinary (non-sensor) collider.
- You can attach a collider to a dynamic bundle for an entity.  These colliders will then be updated by the physics simulation if the entity in question has a rigid body.  Here is an example:
```rust,noplayground
			
	let collider = Collider::new(SharedShape::ball(0.35));
    world
        .insert(helmet, (collider, RigidBody::default()))
        .unwrap();
```
- You can create a sensor collider and use it separately without inserting it into the world, using Collider::new and using
ray casting.

The position of a collider which has an associated rigid body will be set based on the relative position of the entity itself.  If
you create and maintain a collider separate from a rigid body, you will want to set its position using the methods provided within
rapier3d itself.

Colliders are often used to prevent an object from flying off into space or falling forever.  To do this, certain objects within the
scene which other objects may contact due to falling or moving need to be **fixed rigid bodies**, which will not be moved by collision
by another rigid body.  These are objects such as the floor, ceiling or walls of a room.

To specify the shape of your collider, you will want to examine the [SharedShape](https://docs.rs/rapier3d/latest/rapier3d/geometry/struct.SharedShape.html)
struct and its implementation within the Rapier3d library.  There are many options available include ball shaped, cuboids, cones,
capsules and various types of mesh.

As suggested above, a collider can be specified using a complex mesh.  Review the source of the `asset_importer` module to find out more.

