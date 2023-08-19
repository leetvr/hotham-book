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
