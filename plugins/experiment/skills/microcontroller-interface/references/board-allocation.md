# Controller board allocation

Decides which controller board a firmware module belongs on, and whether a new board is needed at all. The decision
lives at the slmc level because it is a firmware-layout decision, and per-system binding-class needs inform it. Loaded
on demand from `microcontroller-interface`'s SKILL.md, which owns the module conventions these boards carry.

---

## When to add to an existing controller board

Default to adding new modules to an existing board. Reasons to consolidate:

- **Pin budget headroom**: Each board has a finite pin count (check the target board's spec sheet). Each module consumes
  1-3 pins. Confirm the target board's free pin count exceeds the new module's pin requirements before adding it.
- **Bandwidth headroom**: Each board's serial bandwidth depends on its USB or UART configuration. Boards running few
  high-rate (sub-millisecond polling) sensors generally have bandwidth headroom, and a high-rate sensor on an
  already-saturated board may need its own board.
- **Role coherence**: The new module shares the same input/output role as the board's existing modules. Mixing input and
  output modules on one board is permitted but reduces debuggability, because emergency resets driven by sensor-side
  keepalive lapses also reset the actuators on the same board.

---

## When to add a new controller board

Stand up a new board (= new target macro in `main.cpp`) when one of these applies:

1. **Interrupt isolation**: The new module needs exclusive control over hardware interrupts (e.g., the `EncoderModule`'s
   `ENCODER_USE_INTERRUPTS` flag is incompatible with any other `AttachInterrupt()`-using library on the same board).
   The current slmc deployment's `ENCODER` target exists precisely for this reason.

2. **Reset isolation**: The new module's correct operation must not be interrupted if another module on the same board
   causes a keepalive-triggered emergency reset. Actuators that must hold state reliably (e.g., a long-running brake
   engagement) belong on a board separated from high-frequency sensors whose polling could lapse the keepalive.

3. **Latency budget**: Multiple polling-style sensors on one board share the `RuntimeCycle()` iteration budget. If the
   new module requires sub-100us polling and the existing board already runs several polling sensors, the cumulative
   cycle time may exceed the budget. Split onto a dedicated board.

4. **Pin or bandwidth exhaustion**: An existing board has run out of physical pins for the new module's requirements, or
   the board's USB serial bandwidth is saturated by existing high-rate data. (In practice, the boards currently in slmc
   do not saturate their serial bandwidth from the existing module set, so this constraint only triggers for
   hypothetical extreme cases.)

5. **Role separation policy**: The acquisition system's architecture deliberately partitions modules by role for
   debuggability or safety. The current slmc deployment's `ACTOR` / `SENSOR` / `ENCODER` split is the canonical example.
   `ACTOR` holds outputs, `SENSOR` holds inputs, and `ENCODER` is dedicated because of constraint #1.

---

## The current slmc deployment

`slmc/src/main.cpp` defines exactly three target macros, `ACTOR` (`kControllerID` 101), `SENSOR` (`kControllerID` 152),
and `ENCODER` (`kControllerID` 203). `ACTOR` instantiates the `wheel_brake` brake, the `reward_valve` and
`gas_puff_valve` valves, and the `screen_trigger` screen trigger. `SENSOR` instantiates the `mesoscope_frame` TTL input,
the `lick_sensor` lick sensor, and the `torque_sensor` torque sensor, and `ENCODER` instantiates the single
`wheel_encoder`. These names, ids, and module assignments are one layout rather than a mandatory one, so a new
acquisition system is free to re-partition. `mesoscope:mesoscope-vr` documents the rig that this deployment currently
drives.

The targets above, and the modules assigned to them, are **reusable assets**. The whole stack is designed for reuse, and
reuse is heavily preferred over standing up new firmware. When bringing up a new acquisition system, prefer, in order:

1. **Reuse an existing target unchanged** when its module set already covers the new system's needs, so the same
   firmware binary drives a different rig with no rebuild.
2. **Reuse existing modules on an existing target**, adding an already-defined module (or a new Python wrapper around
   one, per [Adding modules](../SKILL.md#adding-modules)) to a target that still has pin and bandwidth headroom.
3. **Add a new target** only when the criteria in [When to add a new controller
   board](#when-to-add-a-new-controller-board) force a split no existing target can absorb.

A new system might therefore reuse all three targets, reuse one, or add a fourth, driven by the principles above and a
reuse-first bias rather than by copying or discarding the existing layout.

---

## Workflow: adding a new controller board

1. **Pick a target macro name**: short, all-caps, semantically meaningful (e.g., `STIMULUS`, `RECORD`). The macro is
   conventionally one word, so avoid underscores or punctuation. Document the macro's purpose in a comment on the
   `#elif defined <NAME>` line in `main.cpp`.

2. **Allocate a controller ID**: `uint8_t`, and it must be unique across every controller board that a single
   DataLogger ingests. The ataraxis advised range for `MicroControllerInterface` instances is 101-150. The current slmc
   deployment uses 101, 152, and 203, so two of its three ids sit outside that advised range
   (the per-target `kControllerID` constants in `slmc/src/main.cpp`). Pick a value that no slmc target already uses
   and that does not collide with the advised ranges of other ataraxis libraries, such as video systems. Coordinate
   with the binding-class layer in sle.

3. **Update `main.cpp`**:
   - Add the new `#elif defined <NEW_TARGET>` block.
   - Set `kControllerID` to the chosen value.
   - Include only the headers for modules instantiated on this board.
   - Build the per-board `Module* modules[]` array.

4. **Add the PlatformIO environment**: Add an `[env:<board>_<target>]` section to `platformio.ini` that extends the
   board's `[<board>_base]` template and appends `-D <NEW_TARGET>` to `build_flags`. Without it the target compiles
   only when the macro is passed by hand, so `pio run` never gates it.

5. **Extend the trailing `#else static_assert(false, ...)` block**. It MUST remain the last branch, and its message
   MUST name the new macro alongside the existing ones (the `#else` `static_assert` block in `slmc/src/main.cpp`).
   A build that then selects no target fails with a list of the targets it could have selected.

6. **Update slmc README and CLAUDE.md**: Add the new target to the README's "Per-Target Configuration" bullet list and
   to CLAUDE.md's build-system environment table. The README and CLAUDE.md SHOULD list every supported target.

7. **Hand off to the consuming system's skill**: The host-PC binding class must add a `MicroControllerInterface`
   instance for the new board, carrying the new controller ID and the matching `ModuleInterface` instances. This skill
   does not cover that step. See `mesoscope:mesoscope-vr` for the current worked example.
