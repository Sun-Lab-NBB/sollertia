# GIMBL controllers subsystem

Class-by-class reference for `Assets/Gimbl/Scripts/Controllers/`. Load this when subclassing `ControllerObject`,
touching the shared movement buffer, or extending the `ControllerTypes` enum. `SKILL.md` owns the directory map, the
assembly layout, and the do-not-modify table.

---

## `Gimbl.ControllerObject` and `ControllerOutput`

Abstract base for input devices that drive actor movement. The only runtime subclasses are:

| Class                      | Driven by                     | Purpose                                  |
|----------------------------|-------------------------------|------------------------------------------|
| `LinearTreadmill`          | MQTT `Motion` topic           | Production treadmill input from hardware |
| `SimulatedLinearTreadmill` | Unity Input System (keyboard) | Developer testing without hardware       |

`SimulatedLinearTreadmill` **extends** `LinearTreadmill` but hides its private `Start()` and does not chain. The
lifecycle contract relies on Unity dispatching `Start` per most-derived type only. A third controller subclass follows
the same pattern, hiding through its own `Start` rather than chaining. The non-chaining note in `LinearTreadmill.cs` is
load-bearing for the contract.

**Thread-safety contract on the movement accumulator.** `ControllerObject.movement` is a `ValueBuffer` written from two
threads. `LinearTreadmill.OnMessage` runs on the MQTT message-dispatch thread (MQTTnet background), while
`LinearTreadmill.ProcessMovement` runs on the Unity main thread via `Update`. Both `OnMessage` and `ProcessMovement`
take a `lock (movement)` around the buffer mutation. Any future controller subclass that touches `movement` must take
the same lock, or it will race with the MQTT callback and drop or double-count movement samples.

**`SimulatedLinearTreadmill.MovementSpeedMultiplier = 8.0f`** is hardcoded as a private `const float`. Anyone tuning
keyboard-driven testing speed must edit this constant in `SimulatedLinearTreadmill.cs`. There is no inspector field,
Task Parameters entry, or MQTT topic to override it at runtime. The interaction trigger is wired to `_input.Player.Jump`
(spacebar).

**`ControllerOutput` indirection**: `ActorObject.Controller` is typed as `ControllerOutput` rather than the concrete
`ControllerObject` subclass so swapping controllers does not invalidate the scene's serialized reference.
`MainWindow.EnsureControllers` attaches one `ControllerOutput` per generated controller GameObject and sets `master` to
the sibling controller component. The indirection erases the subclass distinction at the Inspector field level.

---

## `Gimbl.ControllerTypes`

Enum of supported controller subclasses (`LinearTreadmill`, `SimulatedLinearTreadmill`).
`MainWindow.BuildControllerSpecs` resolves each enum value to its runtime `Type` via reflection
(`controllerAssembly.GetType($"Gimbl.{enumName}")`, where `controllerAssembly` is `typeof(ControllerObject).Assembly`)
once at type init, and maps each to a display name via a small switch (`LinearTreadmill → "Linear"`,
`SimulatedLinearTreadmill → "Simulated Linear"`). An unresolved value logs `Unable to resolve the controller type for
the <EnumName> member. The ControllerObject assembly must declare a 'Gimbl.<EnumName>' class, but it declares none.`
once, not once per scene. Adding a new controller class requires:

1. Subclass `ControllerObject` (and hide `Start` per the non-chaining contract if the subclass adds an MQTT
   subscription). The class MUST be declared in `namespace Gimbl` under `Assets/Gimbl/Scripts/`, so it lands in the
   `Sollertia.Gimbl` assembly the reflection lookup searches.
2. Add a value to `ControllerTypes`.
3. Optionally add a display-name case to `BuildControllerSpecs` if the default `ToString` is not user-friendly.
4. Update `Assets/Tests/EditMode/ControllerTests.cs`, which pins the enum shape:
   `ControllerTypes_MemberSet_HoldsExactlyTwoNamesInDeclarationOrder` asserts the member count and the name array,
   `ControllerTypes_MemberOrdinals_MatchTheDeclaredOrder` asserts each ordinal, and
   `ControllerTypes_EveryMemberName_ResolvesToAControllerObjectSubclass` enforces the `Gimbl.<EnumName>` naming
   contract. See `/unity-tests`.

`MainWindow.EnsureControllers` will pick up the new entry on the next scene init.
