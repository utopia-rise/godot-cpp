# godot-jvm divergence log

This branch, `godot-jvm`, is the [Godot-JVM](https://github.com/utopia-rise/godot-jvm) fork of
`godot-cpp`. Godot-JVM consumes it as a submodule pointed at this branch, not at upstream
`godotengine/godot-cpp` and not at this fork's `master`. It customizes godot-cpp's generated-binding
behavior for Godot-JVM's own object-tracking model; it is not a collection of upstream bug fixes.

The branch is rebased/rebuilt from the upstream tag it tracks, currently `godot-4.5-stable`
(`60b5a4196de8442b43b32ba68ebe1e79cfcb762f`). Every commit added on top of that tag must be logged
here, in commit order, with its hash and a short explanation of what it changes and why. A commit
that only edits this file is the one exception, since its entry could not name its own hash.

The log is appended in the same commit as the change it describes, so the two travel together;
the Godot-JVM repository then moves its submodule pointer to that commit.

## Commits on `godot-jvm`

### `43234d42749378830bff939b41aa52adf2c70d39` — expose ScriptExtension's instance-create virtuals as raw object pointers

Adds a `VIRTUAL_RAW_OBJECT_ARGS` opt-out set to `binding_generator.py` and
honours it in `make_signature()`, so `ScriptExtension::_instance_create` and
`_placeholder_instance_create` are generated taking `GodotObject *` instead
of `Object *`.

`Object::set_script()` is the only caller of those virtuals, so every object
that ever gets a JVM script attached — including every node of every scene
loaded at runtime — passes through them. Decoding an `Object *` parameter
runs `PtrToArg<T *>::convert()`, which calls `get_object_instance_binding()`
and therefore builds, and permanently registers, a godot-cpp wrapper for an
object this module already tracks itself (see `RawObject` in
`cpp/engine/godot_object.h`); `JvmScript` unwrapped it again on the very next
line.

Nothing else in godot-cpp needed changing: `GodotObject` is `typedef void`,
so `PtrToArg<GodotObject *>` resolves to the `PtrToArg<void *>`
specialization from `GDVIRTUAL_NATIVE_PTR(void)`, which reads the pointer
straight out of the argument slot, and `BIND_VIRTUAL_METHOD` keeps working
untouched. It is an explicit per-method opt-out rather than a blanket rule,
because everywhere else the wrapper is what makes the C++ API usable.

Regenerate (`gen/` is gitignored, so a clean build picks this up
automatically) after changing the set.

### `2d8ecb5c4c0019996e3a3f6319a2966dd9ec0ffa` — generate `Callable::get_object()` and `Signal::get_object()` as raw object pointers

Adds a `BUILTIN_RAW_OBJECT_RETURNS` opt-out set to `binding_generator.py`,
the builtin-class counterpart of `VIRTUAL_RAW_OBJECT_ARGS`, and honours it in
three places: the builtin header declaration, the builtin source body and
`make_signature()`. The two accessors are generated returning
`GodotObject *`, read straight out of the ptrcall return slot with
`_call_builtin_method_ptr_ret<GodotObject *>`, instead of the `Object *`
wrapper that `_call_builtin_method_ptr_ret_obj()` looks up, and permanently
registers, through `get_object_instance_binding()`.

This module never wants that wrapper: the `Signal` wire row and the Callable
bridge hand the object to the JVM as a `RawObject`, and until now had to go
through `get_object_id()` plus `RawObject::from_instance_id()`, an ObjectDB
lookup, to avoid materializing one. Both callers now use `get_object()`
directly. `get_object_id()` is unchanged for callers that want the id.

Builtin headers that return the raw type gain a `typedef void GodotObject;`
line next to their forward declarations, identical to the one in
`classes/wrapped.hpp`, so they take on no new include. `gen/` is gitignored
and regenerates on the next build.
