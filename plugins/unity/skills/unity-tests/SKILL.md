---
name: unity-tests
description: >-
  Documents the sollertia-virtual-reality Unity Test Framework suite and the project's eight-assembly
  .asmdef layout. Covers the EditMode / PlayMode placement rule, the Support helpers, the fixtures
  that pin enum, topic, protected-asset, and bridge-tool contracts, and both run paths. Use when
  adding or modifying a C# script or test, when a change breaks a fixture, or when deciding which
  assembly a new script folder joins.
user-invocable: false
---

# Sollertia Unity test suite

Documents the Unity Test Framework suite under `Assets/Tests/` and the assembly definitions every
`sollertia-virtual-reality` script compiles into.

**Reference-only skill.** No upstream. Agents arrive here from `/task-generator`, `/zone-prefabs`,
`/mqtt-contract`, `/gimbl-framework`, `/unity-mcp-environment-setup`, and `/play-mode` whenever a C#
change needs a matching fixture, a new script
folder needs an assembly, or a fixture that pins a contract has started failing.

---

## Scope

**Covers:**
- The three test assemblies and the rule deciding whether a test belongs in EditMode or PlayMode
- The `Assets/Tests/Support/` helper inventory and what each helper supplies
- The eight-assembly `.asmdef` catalog, its platform and define constraints, and the rule a new
  script folder follows
- Running the suite from the Test Runner window and headlessly, including the Editor-lock constraint
- The fixtures that pin enum, topic, protected-asset, and bridge-tool contracts, so an agent knows
  what a change will break

**Does not cover:**
- C# formatting, naming, and XML documentation conventions (see `automation:csharp-style`)
- Interactive Play Mode driven through the bridge tools (see `/play-mode`)
- The `CreateTask` generation pipeline the generator fixtures exercise (see `/task-generator`)
- Zone prefab authoring and the `TriggerMode` extension workflow (see `/zone-prefabs`)
- The topic catalog itself and its payload shapes (see `/mqtt-contract`)
- The GIMBL class APIs the runtime fixtures drive (see `/gimbl-framework`)
- The bridge architecture and the per-tool ownership table (see `/unity-mcp-environment-setup`)

---

## Test assemblies

| Assembly                   | Folder                   | Holds                                        |
|----------------------------|--------------------------|----------------------------------------------|
| `Sollertia.Tests.EditMode` | `Assets/Tests/EditMode/` | 31 fixtures driven without a player loop     |
| `Sollertia.Tests.PlayMode` | `Assets/Tests/PlayMode/` | 4 fixtures driven under the real player loop |
| `Sollertia.Tests.Support`  | `Assets/Tests/Support/`  | 10 helper types both test assemblies draw on |

**EditMode** drives the private Unity lifecycle callbacks (`Awake`, `Start`, `Update`,
`OnTriggerEnter`, `OnTriggerExit`) through the Support assembly's `PrivateAccess` reflection helper,
which keeps every transition deterministic and free of frames and physics. It holds three groups: the
runtime state machines, the editor-only surface (`CreateTask`, `McpBridge`, `MiniJson`, `MainWindow`,
`Monitor`, and the full-screen view classes), and the pure schema and serialization classes
(`ConfigLoader`, `TaskTemplate`, `Cue`, `TrialStructure`, `VREnvironment`, `MQTTTopics`). The
`CreateTask`, `McpBridge`, `MiniJson`, and `MainWindow` fixtures can only live here, because
`Sollertia.Tests.EditMode` is the
only test assembly referencing `Sollertia.Gimbl.Editor` and `Sollertia.InfiniteCorridorTask.Editor`.
`Monitor` and the full-screen view classes compile into `Sollertia.Gimbl` behind `#if UNITY_EDITOR`
guards, so they join the same group for their editor-only surface rather than for an assembly
reference.

**PlayMode** holds the tests needing something Edit Mode cannot supply: real frames, the trigger
callbacks Unity's own physics raises against a Rigidbody-carrying actor, real elapsed wall-clock time,
and the engine-invoked `Awake` / `OnEnable` / `Start` / `OnDestroy` ordering.

| PlayMode fixture          | Covers                                                                                |
|---------------------------|---------------------------------------------------------------------------------------|
| `McpBridgePlayModeTests`  | `enter_play_mode`, reaching its already-playing branch from inside the player loop    |
| `MqttClientPlayModeTests` | `MQTTClient`, `MQTTChannel`, `MQTTConnectorObject`, and `LickStimulusSpawner`         |
| `TaskPlayModeTests`       | `Task` corridor advance across real frames, plus engine-imposed callback ordering     |
| `ZonePlayModeTests`       | Physics-raised trigger callbacks, `Stopwatch` occupancy timing, and `Awake` / `Start` |

You MUST place a new test in EditMode unless it needs one of the four PlayMode conditions above.
A PlayMode fixture is slower and reaches the same state machines the EditMode fixtures already pin,
so it is reserved for the behavior reflection cannot observe.

---

## Support helpers

`Sollertia.Tests.Support` compiles for every platform and is referenced by both test assemblies, so a
helper is written once and used from either mode.

| Helper              | Purpose                                                                                          |
|---------------------|--------------------------------------------------------------------------------------------------|
| `PrivateAccess`     | Reads, writes, and invokes non-public members, including private Unity lifecycle callbacks       |
| `TemplateWorkspace` | Stages a throwaway `Configurations/` and `Textures/` directory pair under the system temp root   |
| `TemplateYaml`      | Renders a complete task template document, with each top-level section individually suppressible |
| `TrialYaml`         | Renders one `trial_structures` entry, omitting the YAML key for any field set to null            |
| `CueYaml`           | Renders one `cues` entry, omitting the YAML key for any field set to null                        |
| `VrEnvironmentYaml` | Renders the `vr_environment` geometry block, omitting the key for any field set to null          |
| `YamlScalar`        | Renders numbers, strings, and booleans as invariant YAML scalars, including `.nan` and `.inf`    |
| `MqttTestHarness`   | Installs an `MQTTClient` singleton and captures every payload published on every known topic     |
| `ZoneRig`           | Assembles a `Task` plus trigger zone hierarchy and exposes the transitions a test drives         |
| `ZoneRigOptions`    | Selects which zone components a `ZoneRig` assembles and the field values they start from         |

`TemplateWorkspace` reproduces the two-directory shape `ConfigLoader` requires, because a cue texture
resolves relative to the template file as `<template directory>/../Textures`.

`ZoneRig` mirrors the hand-authored `StimulusTriggerZone.prefab` and `OccupancyTriggerZone.prefab`
hierarchies, because every zone resolves its collaborators through `GetComponentInChildren` or
`GetComponentInParent` at `Start`. Its drive methods (`StartComponents`, `Tick`, `EnterStimulusZone`,
`ExitOccupancyZone`, `RaiseInteraction`, and the rest) advance the state machine one deterministic
step at a time.

`MqttTestHarness` contacts no broker. `MQTTClient.Publish` routes to in-process subscribers whenever
the client is disconnected, so a capture channel observes exactly what the production publish path
produced and a harness publish reaches the real listener the code under test registered.
`MqttTestHarness.KnownTopics()` reflects over the `MQTTTopics` literals, so a new topic constant gains
a capture channel in every fixture with no harness edit.

`TemplateWorkspace`, `MqttTestHarness`, and `ZoneRig` are `IDisposable` and are created through a
static `Create` factory, so a fixture holds each one in a `using` block or disposes it in `TearDown`.

---

## Assembly definitions

Every script compiles into a named assembly declared by an `.asmdef`, because a test assembly is
unable to reference Unity's predefined `Assembly-CSharp`. A new script folder joins an existing
assembly by sitting inside its subtree, or declares its own `.asmdef` and is referenced from every
assembly consuming it, including `Sollertia.Tests.Support`, `Sollertia.Tests.EditMode`, and
`Sollertia.Tests.PlayMode`, or the tests cannot see the type.

| Assembly                                | Folder                                        | References                                                 |
|-----------------------------------------|-----------------------------------------------|------------------------------------------------------------|
| `Sollertia.Gimbl`                       | `Assets/Gimbl/Scripts/`                       | `Unity.InputSystem`                                        |
| `Sollertia.Gimbl.Editor`                | `Assets/Gimbl/Editor/`                        | Gimbl, InfiniteCorridorTask                                |
| `Sollertia.InfiniteCorridorTask`        | `Assets/InfiniteCorridorTask/Scripts/`        | Gimbl                                                      |
| `Sollertia.InfiniteCorridorTask.Editor` | `Assets/InfiniteCorridorTask/Scripts/Editor/` | Gimbl, Gimbl.Editor, corridor runtime                      |
| `Sollertia.UI`                          | `Assets/UI-lick-reward/`                      | Gimbl, InfiniteCorridorTask                                |
| `Sollertia.Tests.Support`               | `Assets/Tests/Support/`                       | every runtime assembly, UnityEngine.TestRunner             |
| `Sollertia.Tests.EditMode`              | `Assets/Tests/EditMode/`                      | every assembly above, both TestRunner assemblies           |
| `Sollertia.Tests.PlayMode`              | `Assets/Tests/PlayMode/`                      | runtime assemblies and Support, both TestRunner assemblies |

`Sollertia.Gimbl.Editor`, `Sollertia.InfiniteCorridorTask.Editor`, and `Sollertia.Tests.EditMode`
declare `"includePlatforms": ["Editor"]`. All three test assemblies declare
`"defineConstraints": ["UNITY_INCLUDE_TESTS"]` and `"autoReferenced": false`, so none of them compiles
into a player build. `MQTTnet.dll` and `YamlDotNet.dll` are auto-referenced plugins for the production
assemblies, and the three test assemblies set `"overrideReferences": true` with
`"precompiledReferences": ["nunit.framework.dll", "MQTTnet.dll", "YamlDotNet.dll"]` to pick up NUnit
alongside them.

`McpBridge.cs` sits in `Sollertia.InfiniteCorridorTask.Editor`, which is why
`Sollertia.Tests.PlayMode` (whose reference list excludes both editor assemblies) resolves the bridge
type by name through reflection rather than a direct `using`.

---

## Running the suite

Run the suite from `Window → General → Test Runner`, which exposes the EditMode and PlayMode tabs
separately. The PlayMode tab drives its own Play Mode transitions, so you MUST NOT interleave a
PlayMode run with `enter_play_mode_tool` or `exit_play_mode_tool` (see `/play-mode`).

Both platforms also run headlessly:

```bash
Unity -batchmode -nographics -projectPath . -runTests -testPlatform EditMode -testResults editmode-results.xml
Unity -batchmode -nographics -projectPath . -runTests -testPlatform PlayMode -testResults playmode-results.xml
```

Unity holds a per-project lock, so a headless run requires the Editor closed on that project. Closing
the Editor also takes the `McpBridge` HTTP relay offline, which means every agentic Unity tool in this
plugin is unavailable for the duration of the run. You SHOULD ask the user to run the suite from the
Test Runner while the Editor stays open, or to copy the project directory and run headlessly against
the copy, which keeps the Editor session and the bridge alive.

Both platforms passing is a pre-commit gate for `sollertia-virtual-reality`, alongside
`csharpier format .`.

---

## Contract-pinning fixtures

These fixtures assert a declared set rather than a behavior, so extending the set fails them by
design. You MUST update the matching fixture in the same change as the source edit.

| Fixture             | Pins                                                                                       | Update when                               |
|---------------------|--------------------------------------------------------------------------------------------|-------------------------------------------|
| `MQTTTopicsTests`   | `ExpectedTopicCount = 12` and the twelve-literal `ExpectedTopics` array                    | An `MQTTTopics` constant is added         |
| `TriggerModeTests`  | Five `TriggerMode` members, their declaration order, and the accepted ordinals 0 through 4 | A `TriggerMode` member is added           |
| `ControllerTests`   | `ControllerTypes` holding exactly two members, and each name resolving to `Gimbl.<Name>`   | A controller type is added                |
| `McpBridgeTests`    | Eleven of the fifteen dispatched tool names, and seven protected asset paths               | A bridge tool or protected asset is added |
| `ConfigLoaderTests` | The accepted `trigger_type` literal set quoted in the rejection message                    | A `trigger_type` literal is accepted      |

**`MQTTTopicsTests`** requires three edits for a new topic: bump `ExpectedTopicCount`, add the literal
to `ExpectedTopics`, and add a `<Topic>_Constant_EqualsTheContractLiteral` test matching the existing
per-topic pattern. The shared structural helper re-asserts the count, so a stale
`ExpectedTopicCount` fails every structural test in the fixture at once. A topic whose literal differs
from its field name fails `DeclaredTopics_EveryFieldName_EqualsItsLiteralValue`, and a literal
carrying `/`, `#`, `+`, or whitespace fails the separator and whitespace tests.

**`TriggerModeTests`** requires four edits for a new member: bump the count in
`GetValues_TriggerModeEnumeration_DeclaresExactlyFiveMembers`, extend the `expected` array in
`GetNames_TriggerModeEnumeration_MatchesTheDeclaredOrder`, extend the accepted range in
`IsDefined_OrdinalsAroundTheDeclaredRange_AcceptsZeroThroughFourOnly`, and add a
`<Member>_SerializedOrdinal_Is<N>` test. Rename the two test methods whose names spell the count and
the range, because those names encode the numbers they assert.

**`ControllerTests`** pins the `ControllerTypes` contract that `MainWindow.BuildControllerSpecs`
depends on, because `EnsureControllers` instantiates one controller GameObject per enum member from
the resolved spec table: `ControllerTypes_MemberSet_HoldsExactlyTwoNamesInDeclarationOrder` fixes the count and the
order, and `ControllerTypes_EveryMemberName_ResolvesToAControllerObjectSubclass` requires a
`Gimbl.<MemberName>` type deriving from `ControllerObject` for every member.

**`McpBridgeTests`** carries the per-tool registration a new bridge tool joins.
`Dispatch_DeclaredToolName_DoesNotFallThroughToUnknownTool` takes one `[TestCase]` per tool name and
currently covers eleven of the fifteen names `McpBridge.Dispatch` handles. Its XML remark states that
count and names the four exclusions, so the remark is updated alongside the case list. The exclusions
are deliberate: `enter_play_mode` would strand the Editor in Play Mode for the rest of the run and is
covered by `McpBridgePlayModeTests` instead, while `read_task_parameters`, `write_task_parameters`,
and `refresh_monitors` need the `FullScreenViewManager` fixture and are covered by
`McpBridgeTaskParametersTests`. A new tool joins whichever of the three fixtures its handler's
prerequisites allow. `DeleteAsset_ProtectedHandAuthoredAsset_RefusesWithoutDeletingIt` takes one
`[TestCase]` per entry in `McpBridge.DeleteProtectedPaths` except the experiment template scene,
which `IsDeleteAllowed_UnsafeOrUnlistedPath_ReturnsFalse` pins instead, so a new hand-authored asset
adds a case to one of the two.

**`ConfigLoaderTests`** asserts the accepted-literal substring inside the message
`LoadTemplate_UnknownTriggerType_ThrowsInvalidData` expects, so a new `trigger_type` literal both
extends that substring and earns a `LoadTemplate_<NewMode>TriggerType_...` acceptance test.

---

## Authoring conventions

- **Namespace and file naming.** A fixture is named `<TypeUnderTest>Tests`, lives in the namespace its
  assembly's `rootNamespace` declares, and carries `[TestFixture]` plus a `<summary>` naming the class
  it verifies.
- **Test method naming.** Fixtures spell `Member_Condition_ExpectedOutcome`, as in
  `OnMessage_NegativeMovement_AddsTheNegativeValue`.
- **Project-mutating fixtures namespace their assets.** `CreateTaskTests` prefixes every template it
  writes with `ZZTest_` and every cue with `ZZ`, and `TagsAndLayersTests` prefixes every tag and layer
  with `ZZTest`, because both write into the live project rather than a temp directory. A new fixture
  touching the `AssetDatabase` or `TagManager` adopts the same prefix and removes every prefixed
  artifact in teardown.
- **Reach private members through `PrivateAccess`.** You MUST NOT widen a member's access modifier to
  make it testable.
- **Assert against a written-out literal, not the constant under test.** `MQTTTopicsTests` writes each
  expected topic out explicitly, because reading the constant back makes the assertion tautological
  and lets a rename travel silently into `sollertia-experiment`.
- **CSharpier formats test files too.** Run `csharpier format .` over the whole project before
  committing.

---

## Related skills

| Skill                                        | Relationship                                                                                  |
|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| `/task-generator` (this plugin)              | Owns the `CreateTask` pipeline that `CreateTaskTests` and the schema fixtures gate            |
| `/zone-prefabs` (this plugin)                | Owns the zone extension workflow whose `TriggerMode` and protected-asset edits break fixtures |
| `/mqtt-contract` (this plugin)               | Owns the topic catalog `MQTTTopicsTests` pins                                                 |
| `/gimbl-framework` (this plugin)             | Owns the GIMBL types the runtime and controller fixtures drive                                |
| `/unity-mcp-environment-setup` (this plugin) | Owns the bridge architecture and the workflow for adding a bridge tool                        |
| `/play-mode` (this plugin)                   | Consumer of the same Editor. Its bridge tools MUST NOT overlap a PlayMode run                 |
| `automation:csharp-style`                    | Required when authoring a fixture, a helper, or the code under test                           |
| `automation:commit`                          | Run after both platforms pass and CSharpier is clean                                          |

---

## Verification checklist

You MUST verify this checklist before submitting any C# change to `sollertia-virtual-reality`.

```text
Unity Test Suite Compliance:
- [ ] The new or changed behavior has a fixture, placed in Assets/Tests/EditMode/ unless it needs real
      frames, physics-raised trigger callbacks, real elapsed time, or engine-invoked callback ordering
- [ ] Private lifecycle callbacks are driven through PrivateAccess rather than by widening access
- [ ] Every IDisposable Support helper the fixture creates is held in a using block or disposed in TearDown
- [ ] A fixture writing into the live AssetDatabase or TagManager prefixes its artifacts with ZZTest and
      removes every prefixed artifact in teardown
- [ ] A new MQTTTopics constant updated ExpectedTopicCount, ExpectedTopics, and the per-topic literal test
      in MQTTTopicsTests
- [ ] A new TriggerMode member updated the count, name array, ordinal range, and per-member ordinal test
      in TriggerModeTests, including the two method names that spell the count and the range
- [ ] A new ControllerTypes member updated ControllerTests and ships a Gimbl.<MemberName> ControllerObject
      subclass
- [ ] A new McpBridge.Dispatch case joined the [TestCase] list on McpBridgeTests, McpBridgePlayModeTests, or
      McpBridgeTaskParametersTests, and the McpBridgeTests remark's covered-tool count was updated
- [ ] A new DeleteProtectedPaths entry added a [TestCase] to DeleteAsset_ProtectedHandAuthoredAsset_
      RefusesWithoutDeletingIt
- [ ] A new trigger_type literal updated the accepted-literal substring in ConfigLoaderTests and gained an
      acceptance test
- [ ] A new script folder either sits inside an existing assembly's subtree or declares its own .asmdef and
      is referenced from every consuming assembly, including the test assemblies
- [ ] A new .asmdef declares rootNamespace, and a test assembly variant declares the UNITY_INCLUDE_TESTS
      define constraint plus the overrideReferences and precompiledReferences block

Tool-settled (run `csharpier check .`, then both Test Runner tabs or the two headless invocations):
- [ ] csharpier format . ran cleanly over the project
- [ ] The EditMode platform passes with no failures
- [ ] The PlayMode platform passes with no failures
```
