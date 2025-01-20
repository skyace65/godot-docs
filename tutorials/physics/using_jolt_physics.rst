.. _doc_using_jolt_physics:

Using Jolt Physics
==================

Overview
--------

The Jolt physics engine was added as an alternative to the existing GodotPhysics3D
physics engine in 4.4. Jolt is developed by Jorrit Rouwe with a focus on games and
VR applications. Previously it was available as a extension but is now built into
Godot.

It is important to note that the built in Jolt Physics module is considered
**not finished**, **experimental**, and **lacks feature partiy** with both
GodotPhysics3D, and the Jolt Extension. Behavior may change as it is developed
further, please keep that in mind when choosing what to use for your project. A
roadmap of this module can be found at the bottom of this page.

The existing extension is now considered in maintenance mode. That means bug fixes
will be merged, and it will be kept compatible with new versions of Godot until
the built-in module has feature parity with the extension. The extension can be
found `here on GitHub <https://github.com/godot-jolt/godot-jolt>`_ and in Godot's asset
library.

To change the 3D physics engine to the Jolt module go to
:ref:`**Project Settings > Physics > 3D > Physics Engine**<class_ProjectSettings_property_physics/3D/Physics_Engine>`
and select ``Jolt Physics`` from the dropdown menu. Once you've done that click the
``Save & Restart`` button, when the editor opens again 3D scenes should now be using
Jolt for physics.

Notable differences to Godot Physics
------------------------------------

There are many differences between the existing GodotPhysics3D engine and Jolt.

Area3D and static bodies
~~~~~~~~~~~~~~~~~~~~~~~~
When using Jolt, :ref:`class_Area3D` will not detect overlaps with :ref:`class_StaticBody3D`
(nor a :ref:`class_RigidBody3D` frozen with ``FREEZE_MODE_STATIC``) by default, for
performance reasons. If you have many/large :ref:`class_Area3D` overlapping with
complex static geometry, such as :ref:`class_ConcavePolygonShape3D` or
:ref:`class_HeightMapShape3D`, you can end up wasting a significant amount of CPU
performance without realizing it.

For this reason this behavior is opt-in through the project setting
:ref:`physics/jolt_physics_3d/simulation/areas_detect_static_bodies<class_ProjectSettings_property_physics/jolt_physics_3d/simulation/areas_detect_static_bodies>`,
with the recommendation that you set up your collision layers/masks in such a way that only
the relevant :ref:`class_Area3D` are able to detect collisions with static bodies.

Joint properties
~~~~~~~~~~~~~~~~

The current interfaces for the 3D joint nodes don't quite line up with the interface
of Jolt's own joints. As such, there are a number of joint properties that are not
supported, mainly ones related to configuring the joint's soft limits.

The unsupported properties are:

- PinJoint3D: ``bias``, ``damping``, ``impulse_clamp``
- HingeJoint3D: ``bias``, ``softness``, ``relaxation``
- SliderJoint3D: ``angular_\*``, ``\*_limit/softness``, ``\*_limit/restitution``, ``\*_limit/damping``
- ConeTwistJoint3D: ``bias``, ``relaxation``, ``softness``
- Generic6DOFJoint3D: ``*_limit_*/softness``, ``*_limit_*/restitution``, ``*_limit_*/damping``, ``*_limit_*/erp``

Currently an error is emitted if you set these properties to anything but their
default values.

Single-body joints
~~~~~~~~~~~~~~~~~~

You can, in Godot, omit one of the joint bodies for a two-body joint and effectively
have "the world" be the other body. However, the node path that you assign your body
to (node_a vs node_b) is ignored. Godot Physics will always behave as if you
assigned it to node_a, and since node_a is also what defines the frame of reference
for the joint limits, you end up with inverted limits and a potentially strange
limit shape, especially if your limits allow both linear and angular degrees of
freedom.

Jolt will behave as if you assigned the body to node_b instead, with node_a
representing "the world". There is a project setting called :ref:`physics/jolt_physics_3d/joints/world_node<class_ProjectSettings_property_physics/jolt_physics_3d/joints/world_node>`
that lets you toggle this behavior, if you need compatibility for an existing project.

Collision margins
~~~~~~~~~~~~~~~~~

Jolt (and other similar physics engines) uses something that Jolt refers to as
"convex radius" to help improve the performance and behavior of the types of
collision detection that Jolt relies on for convex shapes. Other physics engines
(Godot included) might refer to these as "collision margins" instead. Godot exposes
these as the margin property on every Shape3D-derived class, as a leftover from the
Bullet integration in Godot 3, but Godot Physics itself does not use them for
anything.

What these collision margins sometimes do in other engines (as described in Godot's
documentation) is effectively add a "shell" around the shape, slightly increasing
its size while also rounding off any edges/corners. In Jolt however, these margins
are first used to shrink the shape, and then the "shell" is applied, resulting in
edges/corners being similarly rounded off, but without increasing the size of the
shape.

To prevent having to tweak this margin property manually, since its default value
can be problematic for smaller shapes, this module exposes a project setting called
:ref:`physics/jolt_physics_3d/collisions/collision_margin_fraction<class_ProjectSettings_property_physics/jolt_physics_3d/collisions/collision_margin_fraction>`
which is multiplied with the smallest axis of the shape's AABB to calculate the
actual margin. The margin property of the shape is then instead used as an upper
bound.

These margins should, for most use-cases, be more or less transparent, but can
sometimes result in odd collision normals when performing shape queries. You can
lower the above mentioned project setting to mitigate some of this, including
setting it to ``0``, but too small of a margin can also cause odd collision results,
so is generally not recommended.

Compound shapes
~~~~~~~~~~~~~~~

Compound shapes (i.e. bodies with more than one shape) are implemented differently
from Godot Physics.

Jolt offers the choice of two compound shapes, one called MutableCompoundShape and
one called StaticCompoundShape. The former trades in runtime performance for faster
construction/modification time, and vice versa for the latter.

Godot Physics maps closer to Jolt's MutableCompoundShape, but StaticCompoundShape is
used for this implementation, as it simplified things a bit (with being able to
discard and rebuild the whole thing when modified) and more people would likely
benefit from the improved runtime performance as opposed to mutation performance.

To mitigate the cost of this potentially expensive rebuild, shape
changes are only ever "committed" to Jolt when the body has entered into a scene
tree. This means that you can make shape changes (including adding/removing shapes)
very quickly so long as the body isn't attached to the scene tree. However, if you
do in fact add the body to a scene tree, and then start adding/removing/changing
shapes on the body, you can end up with worse performance than Godot Physics. This
is prominently visible in the "Voxel Game" demo project, as one example.

The plan for this is to replace StaticCompoundShape with a custom compound shape
that wraps StaticCompoundShape, but which will always defer its building/committing
only until absolutely necessary, meaning either right before a simulation step or
when performing queries against the body.

Scaling shapes/bodies/queries
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Godot Physics supports scaling the transform of collision shapes, shape queries, as
well as static and kinematic bodies, meaning :ref:`class_StaticBody3D`,
:ref:`class_CharacterBody3D`, :ref:`class_AnimatableBody3D` and frozen :ref:`class_RigidBody3D`.
It does not however support scaling simulated/dynamic bodies, such as a non-frozen :ref:`class_RigidBody3D`,
and will effectively discard any such scaling, instead treating it as (1, 1, 1).

Jolt does however support scaling everywhere, which means that :ref:`class_RigidBody3D`
will support scaling when using Jolt, despite the warning shown on the node
currently.

Jolt also supports non-uniform scaling, so long as the inherent primitive shape is
preserved. For example, you can scale a cylinder along its height axis but not along
its other axes. You will however currently see warnings on the shape node when doing
this, due to Godot Physics not supporting this.

Since invalid scaling can cause a number of weird artifacts, and sometimes outright
crash the simulation, there are runtime error checks that "sanitize" all scaling to
be valid for that particular shape arrangement, and then reports an error if the
difference is above an arbitrary threshold.

Shape-casting
~~~~~~~~~~~~~

Due to Godot having a "safe" and "unsafe" fraction in the results of :ref:`cast_motion<class_PhysicsDirectSpaceState3D_method_cast_motion>`
(and ShapeCast3D), meaning the distances at which the cast shape was found to be
colliding and not colliding respectively, it was not viable to rely on Jolt's own
shape-casting (which uses conservative advancement) to implement :ref:`cast_motion<class_PhysicsDirectSpaceState3D_method_cast_motion>`.
Instead :ref:`cast_motion<class_PhysicsDirectSpaceState3D_method_cast_motion>` is
implemented with a binary search, similar to how Godot Physics does it.

However, in Godot Physics this binary search is hardcoded to 8 steps, which tends to
result in quite jittery output over even moderate distances. With Godot Jolt (and
consequently this module) the number of steps is dynamically calculated  based on
the cast distance, aiming for roughly millimeter precision, and then clamped between
4 and 16 steps.

This does however mean that :ref:`cast_motion<class_PhysicsDirectSpaceState3D_method_cast_motion>`
will technically, when using Jolt, cost more CPU performance the farther you cast
the shape.

Note that this also applies to :ref:`body_test_motion<class_PhysicsServer3D_method_body_test_motion>`, and consequently :ref:`test_move<class_PhysicsBody3D_method_test_move>`,
:ref:`move_and_collide<class_PhysicsBody3D_method_move_and_collide>` and :ref:`move_and_slide<class_CharacterBody3D_method_move_and_slide>`.

Baumgarte stabilization
~~~~~~~~~~~~~~~~~~~~~~~

Jolt employs a technique in its solver called Baumgarte stabilization, which is
meant to mitigate constraint drift within the simulation, resulting in a more stable
simulation. This technique can however result in some artifacts, like piles of
bodies not separating as quickly as one might expect.

The strength of this stabilization can be tweaked using the project setting
:ref:`physics/jolt_physics_3d/simulation/baumgarte_stabilization_factor<class_ProjectSettings_property_physics/jolt_physics_3d/simulation/baumgarte_stabilization_factor>`.
Setting this project setting to ``1.0`` will effectively disable the technique,
resulting in a simulation that more closely resembles Godot Physics, but which is
also more unstable.

Motion queries (move_and_slide, etc.)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Physics servers in Godot are meant to implement the :ref:`PhysicsServer3D.body_test_motion()<class_PhysicsServer3D_method_body_test_motion>`
method, which in turn powers methods like :ref:`PhysicsBody3D.test_move()<class_PhysicsBody3D_method_test_move>`,
:ref:`PhysicsBody3D.move_and_collide()<class_PhysicsBody3D_method_move_and_collide>`
and :ref:`CharacterBody3D.move_and_slide()<class_CharacterBody3D_method_move_and_slide>`,
which are largely meant to be used for moving player characters around.

:ref:`PhysicsServer3D.body_test_motion<class_PhysicsServer3D_method_body_test_motion>`
is split into three parts, the first being depenetration (called "recovery" in
Godot), the second being a shape-cast from the "recovered" position, and the third
being the actual collision check, at the "unsafe" fraction of the shape-cast.

The "recovery" step in Godot Physics is hardcoded to always do 4 iterations with 40%
depenetration per iteration. I figured these constants might be useful to expose, so
when using this module they can be configured in the project settings as
:ref:`physics/jolt_physics_3d/motion_queries/recovery_iterations<class_ProjectSettings_Property_physics/jolt_physics_3d/motion_queries/recovery_iterations>`
and :ref:`physics/jolt_physics_3d/motion_queries/recovery_amount<class_ProjectSettings_Property_physics/jolt_physics_3d/motion_queries/recovery_amount>`
respectively.

The implementation of :ref:`PhysicsServer3D.body_test_motion()<class_PhysicsServer3D_method_body_test_motion>`
in this module also differs slightly from the Godot Physics implementation, as
replicating the Godot Physics version resulted in a prohibitive amount of ghost
collisions for :ref:`move_and_slide<class_CharacterBody3D_method_move_and_slide>`.

The discrepancies are as follows:

- There is no TEST_MOTION_MIN_CONTACT_DEPTH_FACTOR.
- There is no epsilon added to the safe margin during recovery.
- Contacts with normals that are not opposing the motion vector are discarded from
  the final collision check.

While these discrepancies do seem to result in less ghost collisions when using
Jolt, they also introduce new problems:

- :ref:`move_and_slide<class_CharacterBody3D_method_move_and_slide>` is generally
  slower than in Godot Physics, due to :ref:`class_CharacterBody3D`::_snap_on_floor(PR NOTE: Apply_floor_snap?)
  triggering on every call.
- :ref:`class_CharacterBody3D` will not report wall collisions unless you're moving towards them.

It's not clear to me how to fix these issues, but I suspect they will require
changes to move_and_slide on the scene side of things.

Ghost collisions
~~~~~~~~~~~~~~~~

Jolt employs two techniques to mitigate ghost collisions, meaning collisions with
internal edges of shapes/bodies.

The first technique, called "active edge detection", marks edges of triangles in
:ref:`class_ConcavePolygonShape3D` or :ref:`class_HeightMapShape3D` as either "active" or "inactive", based on
the angle to the neighboring triangle. When a collision happens with an inactive
edge the collision normal will be replaced with the triangle's normal instead, to
lessen the effect of ghost collisions.

The angle threshold for this active edge detection is configurable through the
project setting :ref:`physics/jolt_physics_3d/collisions/active_edge_threshold<class_ProjectSettings_property_physics/jolt_physics_3d/collisions/active_edge_threshold>`.

The second technique, called "enhanced internal edge removal", instead adds runtime
checks to detect whether an edge is active or inactive, based on the contact points
of the two bodies. This has the benefit of applying not only to collisions with
:ref:`class_ConcavePolygonShape3D` and :ref:`class_HeightMapShape3D`, but also edges between any shapes within
the same body.

Enhanced internal edge removal can be toggled on and off for the various contexts to
which it's applied, using the ``physics/jolt_physics_3d/*/use_enhanced_internal_edge_removal``
project settings.

Memory usage
~~~~~~~~~~~~

Jolt uses a stack allocator for temporary allocations within its simulation step.
This stack allocator requires allocating a set amount of memory up front, which can
be configured using the :ref:`physics/jolt_physics_3d/limits/temporary_memory_buffer_size<class_ProjectSettings_property_physics/jolt_physics_3d/limits/temporary_memory_buffer_size>`
project setting.

Jolt also makes use of aligned allocations for some of its memory usage, which it
acquires through function pointers that the implementer assigns. Aligned allocations
were recently added to Godot in the form of ``Memory::alloc_aligned_static``,
``Memory::realloc_aligned_static`` and ``Memory::free_aligned_static``, which have all been
hooked up to Jolt. However, these aligned allocation functions don't currently touch
``Memory::mem_usage``, which means that some of Jolt's memory won't be tracked, and thus
the performance monitors in Godot won't be accurate.

Ray-Cast face index
~~~~~~~~~~~~~~~~~~~

The face_index property returned in the results of intersect_ray and RayCast3D will
by default always be ``-1`` with Jolt, the project setting :ref:`physics/jolt_physics_3d/queries/enable_ray_cast_face_index<class_ProjectSettings_property_physics/jolt_physics_3d/queries/enable_ray_cast_face_index>`
will enable them. The reason for this being that Jolt does not store these indices,
and for them to be stored per-triangle userdata must be enabled, which adds about
25% extra memory usage to the underlying Jolt implementation of :ref:`class_ConcavePolygonShape3D`.

Kinematic RigidBody3D contacts
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When using Jolt, a :ref:`class_RigidBody3D` frozen with ``FREEZE_MODE_KINEMATIC`` will by default not
report contacts from collisions with other static/kinematic bodies, for performance
reasons, even when setting a non-zero max_contacts_reported. If you have many/large
kinematic bodies overlapping with complex static geometry, such as :ref:`class_ConcavePolygonShape3D`
or :ref:`class_HeightMapShape3D`, you can end up wasting a significant amount of CPU
performance without realizing it.

For this reason this behavior is opt-in through the project setting
:ref:`physics/jolt_physics_3d/simulation/generate_all_kinematic_contacts<class_ProjectSettings_property_physics/jolt_physics_3d/simulation/generate_all_kinematic_contacts>`.

Contact impulses
~~~~~~~~~~~~~~~~

Jolt is not able to provide any impulse as part of its contact data, due to how it
orders its simulation step, and instead provides a helper function for estimating
what the impulse would be based on various parameters. While this is fine for most
use cases, like emitting a sound based on how hard something collided, it won't be
accurate if a body is colliding with multiple bodies during a simulation step.

Area3D and SoftBody3D
~~~~~~~~~~~~~~~~~~~~~

This module does not currently support any interactions between :ref:`class_SoftBody3D`
and :ref:`class_Area3D`, such as overlap events, or the wind properties found on
:ref:`class_Area3D`. Support for this has been added to Jolt recently, but it is not
currently part of the built-in Jolt module.

ConvexPolygonShape3D
~~~~~~~~~~~~~~~~~~~~

Godot Physics currently skips calculating a proper center-of-mass and inertia for
:ref:`class_ConvexPolygonShape3D`, and instead always uses the shape's local
position as its center of mass, while crudely estimating the inertia. Jolt on the
other hand does calculate a more accurate center-of-mass and inertia for them. As a
result a non-frozen :ref:`class_RigidBody3D` with such a shape in it can behave
differently when comparing the two engines.

WorldBoundaryShape3D
~~~~~~~~~~~~~~~~~~~~

:ref:`class_WorldBoundaryShape3D`, which is meant to represent an infinite plane, is
implemented a bit differently in Jolt compared to Godot Physics. Both engines have
an upper limit for how big the effective size of this plane can be, but this size is
much smaller when using Jolt, in order to avoid precision issues.

You can configure this size using the :ref:`physics/jolt_physics_3d/limits/world_boundary_shape_size<Class_ProjectSettings_Property_physics/jolt_physics_3d/limits/world_boundary_shape_size>`
project setting.

Axis-locking
~~~~~~~~~~~~

The ``PhysicsBody3D.axis_lock_\*`` properties in Godot Physics are implemented by
simply zeroing out the velocities for the selected axes. Jolt instead implements
these by calculating a new inverse mass/inertia, similar to how :ref:`RigidBody3D.lock_rotation<class_RigidBody3D_property_lock_rotation>`
works in Godot Physics, which seems to better mitigate energy loss during
simulation.

However, Jolt does not allow locking all axes, meaning both linear and angular axes,
and an error will be emitted from this module when trying to do so, with the
recommendation that you instead freeze the body entirely. While this is a simple
enough workaround for :ref:`class_RigidBody3D`, you cannot currently freeze a :ref:`class_PhysicalBone3D`, so
if you want to lock all axes of a :ref:`class_PhysicalBone3D` you're forced to resort to calling
:ref:`PhysicsServer3D.body_set_mode()<class_PhysicsServer3D_method_body_set_mode>` yourself.

Notable differences to Godot Jolt
---------------------------------

While this module is largely a straight port of the Godot Jolt extension, with a lot
of cosmetic changes, there are a few things that are different.

Project Settings
~~~~~~~~~~~~~~~~

All project settings have been moved from the ``physics/jolt_3d`` category to
``physics/jolt_physics_3d``.

On top of that, there's been some renaming and refactoring of the individual project
settings as well. These include:

- ``sleep/enabled`` is now ``simulation/allow_sleep.``
- ``sleep/velocity_threshold`` is now ``simulation/sleep_velocity_threshold.``
- ``sleep/time_threshold`` is now ``simulation/sleep_time_threshold.``
- ``collisions/use_shape_margins`` is now ``collisions/collision_margin_fraction``,
  where a value of 0 is equivalent to disabling it.
- ``collisions/use_enhanced_internal_edge_removal`` is now ``simulation/use_enhanced_internal_edge_removal.``
- ``collisions/areas_detect_static_bodies`` is now ``simulation/areas_detect_static_bodies.``
- ``collisions/report_all_kinematic_contacts`` is now ``simulation/generate_all_kinematic_contacts.``
- ``collisions/soft_body_point_margin`` is now ``simulation/soft_body_point_radius.``
- ``collisions/body_pair_cache_enabled is now simulation/body_pair_contact_cache_enabled.``
- ``collisions/body_pair_cache_distance_threshold`` is ``now simulation/body_pair_contact_cache_distance_threshold.``
- ``collisions/body_pair_cache_angle_threshold is now simulation/body_pair_contact_cache_angle_threshold.``
- ``continuous_cd/movement_threshold`` is now ``simulation/continuous_cd_movement_threshold``,
  but expressed as a fraction instead of a percentage.
- ``continuous_cd/max_penetration`` is now ``simulation/continuous_cd_max_penetration``,
  but expressed as a fraction instead of a percentage.
- ``kinematics/use_enhanced_internal_edge_removal`` is now ``motion_queries/use_enhanced_internal_edge_removal.``
- ``kinematics/recovery_iterations`` is now ``motion_queries/recovery_iterations``,
  but expressed as a fraction instead of a percentage.
- ``kinematics/recovery_amount`` is now ``motion_queries/recovery_amount.``
- ``queries/use_legacy_ray_casting`` has been removed.
- ``solver/position_iterations`` is now ``simulation/position_steps.``
- ``solver/velocity_iterations`` is now ``simulation/velocity_steps.``
- ``solver/position_correction`` is now ``simulation/baumgarte_stabilization_factor``,
  but expressed as a fraction instead of a percentage.
- ``solver/active_edge_threshold`` is now ``collisions/active_edge_threshold.``
- ``solver/bounce_velocity_threshold`` is now ``simulation/bounce_velocity_threshold.``
- ``solver/contact_speculative_distance`` is now ``simulation/speculative_contact_distance.``
- ``solver/contact_allowed_penetration`` is now ``simulation/penetration_slop.``
- ``limits/max_angular_velocity`` is now stored as radians instead.
- ``limits/max_temporary_memory`` is now ``limits/temporary_memory_buffer_size.``

There might be some discussion to be had with regards to migrating the settings
values for projects who have previously been relying on the extension.

Joint Nodes
~~~~~~~~~~~

The joint nodes that are exposed in the Godot Jolt extension (JoltPinJoint3D,
JoltHingeJoint3D, JoltSliderJoint3D, JoltConeTwistJoint3D and JoltGeneric6DOFJoint)
have not been included with this module.

Thread Safety
~~~~~~~~~~~~~

Unlike the Godot Jolt extension, this module does have experimental thread-safety,
including support for the :ref:`physics/3d/run_on_separate_thread<class_ProjectSettings_Property_physics/3d/run_on_separate_thread>`
project setting. This is achieved by utilizing the same wrapper server that's used
by Godot Physics, called ``PhysicsServer3DWrapMT``, as well as introducing a mutex
around the rebuilding of shapes, since concurrent shape-casts could otherwise
trigger a race condition.

This has however not been tested very thoroughly, so should be considered
experimental.

Query Performance
~~~~~~~~~~~~~~~~~

The Godot Jolt extension utilizes a custom container (typically referred to as an
"inline vector" or "small vector") in order to avoid heap allocations for physics
queries that return more than a single hit, meaning :ref:`intersect_point<class_PhysicsDirectSpaceState3D_method_intersect_point>`,
:ref:`intersect_shape<class_PhysicsDirectSpaceState3D_method_intersect_shape>`,
:ref:`collide_shape<class_PhysicsDirectSpaceState3D_method_collide_shape>`,
:ref:`get_rest_info<class_PhysicsDirectSpaceState3D_method_get_rest_info>`,
:ref:`cast_motion<class_PhysicsDirectSpaceState3D_method_cast_motion>`,
:ref:`body_test_motion<class_PhysicsServer3D_method_body_test_motion>`,
:ref:`test_move<class_PhysicsBody3D_method_test_move>`, :ref:`move_and_collide<class_PhysicsBody3D_method_move_and_collide>`
and :ref:`move_and_slide<class_CharacterBody3D_method_move_and_slide>`.

For the built-in module ``JPH::Array`` is used for the query collectors instead, which
means that these queries will now always allocate on the heap, likely making them
noticeably slower.

It should be trivial to make some bespoke container for the query collectors that
behaves like an inline/small vector.

Debug Renderer
~~~~~~~~~~~~~~

Jolt provides an interface called ``JPH::DebugRenderer`` for rendering its view of
the physics simulation. In the Godot Jolt extension this is exposed as a custom
:ref:`class_GeometryInstance3D` node called ``JoltDebugGeometry3D``, a different
approach will be used for the built-in module, and for now it is omitted.

Debug snapshots
~~~~~~~~~~~~~~~

The Godot Jolt extension has the ability to generate what Jolt refers to as
"snapshots", where it serializes the state of the physics simulation to a file.
These can then be loaded in Jolt's own Samples application, to debug issues more
closely there, without needing to deal with Godot itself, which has proved to be
quite useful when reporting issues upstream to Jolt.

The code for this is technically included with this module, and can be found as
``JoltPhysicsServer3D::dump_debug_snapshots``, but it is currently unexposed for now.
