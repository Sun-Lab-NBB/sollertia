---
name: external-tool-bindings
description: >-
  Owns the external tool binding convention, which is the admission test that decides whether an outside dependency
  registers with the platform or binds to it across a process boundary. Covers the configuration that addresses the
  tool, the argument vector that invokes it, and the on-disk artifact contract that replaces an API. Use when adding a
  tool that cannot be installed beside the stack, wiring an external invocation into preprocessing or a processing
  stage, or auditing an existing binding against the convention.
user-invocable: false
---

# External tool bindings

Defines how a tool that cannot share the stack's environment reaches the platform, through a process boundary at one
end and an artifact on disk at the other.

You MUST run the admission test in this skill before writing any binding code, and you MUST run the verification
checklist before reporting a binding complete.

---

## Scope

**Covers:**
- The two extension paths, register and bind, and the mechanical admission test that decides between them
- The process boundary, invoked and never imported, with the tool's address resolved at call time
- The configuration section that addresses the tool, and the empty value that disables the binding
- The artifact contract that replaces an API, fixing the artifact's location, naming rule, and shape
- The producer seam in full, and the three obligations it owes the consumer seam
- The producer's degradation rule, and the pointer to the consumer's
- What a binding is denied, the one upstream edit it is permitted, and the places a binding is recorded instead
- The ordered workflow for adding a binding, and the worked instance the platform carries today

**Does not cover:**
- The registry extension path a dependency takes when it does fit the platform vocabulary, which is a pin, an import,
  and the "New `AcquisitionSystems` member" or "New `ReadAssets` member" scenario. Owned by `assets:library-extension`
- The session directory anatomy that receives the artifact, and the `Directories` member naming its subdirectory.
  Owned by `assets:session-data`
- The acquisition-system configuration layer that hosts the binding's configuration section. Owned by
  `/acquisition-system-design`
- The sollertia-experiment extension seams a new acquisition system composes. Owned by `/library-extension`
- The checksum-and-transfer step that carries the artifact, and the preprocessing lifecycle around it. Owned by
  `/data-management`
- The agnostic-worker donation model the consumer seam follows. Owned by `forging:data-processing-design`
- The forging registries a donation joins, the new stage a new artifact kind requires, and the dataset columns the
  worker's output feeds. Owned by `forging:library-extension`
- The donated locator's own contract, and the job discovery and readiness rules it gates. Owned by
  `forging:processing-input-format`
- The worked instance's configuration section. Owned by `mesoscope:mesoscope-vr`
- The worked instance's launcher, join, abort reap, and invocation options. Owned by `mesoscope:mesoscope-vr-runtime`
- The worked instance's consumer stage in detail. Owned by `mesoscope:mesoscope-vr-video-tracking`

---

## Register or bind

An outside dependency takes exactly one of two extension paths, and the admission test below decides which.

| Path       | What the dependency gains                                                                                                     | What it requires                                            |
|------------|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| `register` | A pinned dependency and a direct import, plus an enum member and registry entries when it carries a platform-vocabulary asset | It resolves in the same environment as the stack member     |
| `bind`     | A configuration section, an argument vector, and an artifact on disk                                                          | It runs non-interactively and writes a file the stack reads |

### Blocking conditions

A dependency binds when at least one condition below holds. You MUST record which condition holds and the evidence
proving it, because that evidence is what the binding's documentation later cites. Cite a file and a line when a
manifest on either side declares the conflict. When the condition is a property of the tool's own runtime, license, or
launcher that no manifest states, cite the tool's installation requirement or its vendor documentation instead. Carry
that citation into the README entry and the launcher docstring.

1. **Runtime mismatch.** The tool runs on an interpreter or runtime the stack member does not host, or its declared
   interpreter range and the stack member's `requires-python` are disjoint.
2. **Dependency conflict.** The tool pins a shared library to a range disjoint from the range the stack member pins, so
   no single environment resolves both.
3. **Distribution constraint.** A license, a vendor SDK, or a machine-local installation forbids declaring the tool as
   a dependency of a published package.
4. **Interaction constraint.** The tool runs only under its own session manager, launcher, or graphical shell, so it
   cannot be driven as a library call.

When no condition holds, the dependency registers instead, taking a pin and an import. When it also carries a
platform-vocabulary asset, `assets:library-extension` owns that addition through its "New `AcquisitionSystems` member"
and "New `ReadAssets` member" scenarios.

### Bindability conditions

All three conditions below must hold before a binding is written. A tool failing any of them stays outside the
platform, and its results enter as operator-placed files rather than through a binding.

1. **Non-interactive entry point.** The tool exposes a command invocable as an argument vector, running to completion
   with no terminal interaction.
2. **Predictable output path.** The caller either chooses the output directory or predicts it from the input it passes.
3. **Readable output format.** The stack reads the tool's output with a dependency it already carries, or with one that
   is addable without reintroducing a blocking condition.

---

## The process boundary

A binding is invoked, never imported, and the calling module holds no reference to the bound package.

- The calling module imports nothing from the tool, so a host without the tool installed still imports that module and
  runs every other stage in it.
- The tool's address is configuration rather than a code constant, because the same code runs on a rig that carries the
  tool and on one that does not. The rule admits several shapes. When a named environment the host resolves provides the
  tool, an identity field carries the environment name and the vector spells out the tool's own command token. When the
  host supplies the executable or the runtime root itself, an identity field carries that absolute path, and a tool
  needing a separately installed runtime carries that runtime's root in a second identity field. A tool the host puts
  on `PATH` carries no address field at all. Its vector spells out the bare command token.
- The constants the rule bans are the host-specific ones, meaning filesystem paths, environment names, and installation
  roots. The invocation tokens the vector spells out, meaning the launcher token and the tool's subcommands and flags,
  are platform-owned literals.
- Resolution happens at call time. A host that lacks the tool fails at the call, and only when the binding is enabled.
- The argument vector is a `list[str]` handed to `subprocess` with no shell. Every configuration value is stringified at
  the call site, and every path is passed as `str(path)`.
- An argument whose configuration value is empty is omitted entirely rather than passed as an empty value.
- Interactive output such as a progress bar is suppressed, because the child's stream is captured to a file.
- The launcher's own gate answers whether the host is configured for the tool. The call site may carry a second gate on
  the session's own kind, answering whether this session wants the run at all.

Concurrency is the caller's choice rather than a rule of the convention. A short run is launched synchronously, and the
launcher checks the exit status and the artifact's existence inline before it returns. A long run the caller can overlap
with other work is launched asynchronously, and the launcher then returns a process handle or `None`, so the caller owns
the join and the abort reap.

The vector's shape is a host-owned prefix followed by the tool's own tokens. The prefix changes with the launcher or the
runtime root the host needs, and the rest of the convention survives that change unaltered.

---

## The configuration declaration

The binding is declared in a dataclass section of the acquisition system's configuration, and the mounting of that
section is owned by `/acquisition-system-design`.

Fields split into two kinds:

| Kind        | What it holds                                                                                | Default                  | Effect when empty       |
|-------------|----------------------------------------------------------------------------------------------|--------------------------|-------------------------|
| `identity`  | The tool's address, the runtime root it needs, and where its project, model, or script lives | Empty string or `Path()` | Disables the binding    |
| `parameter` | A knob that tunes an already-enabled run                                                     | A working value          | Omits that one argument |

The launcher runs only when every identity field carries a value, so a section carrying one configured identity field
and one empty field runs nothing. Every field is defaulted, so a freshly generated system configuration launches no
external process. A tool needing no address, no runtime root, and no project path still declares one identity field
naming its command token, so the gate is never vacuous. A binding exposing zero parameter fields is fully conformant,
and the parameter set is chosen by the tool rather than by the convention.

---

## The artifact contract

An artifact on disk replaces the API a registered dependency would expose. The contract fixes three things, and a
binding is incomplete until all three are stated in code.

### Location

The artifact's directory is named through a session-record field on both sides, never as a path literal, and
`assets:session-data` owns the session anatomy those fields resolve. No path is passed between the two seams, because
the session record is the only channel.

The default home is a subdirectory of the session's raw tree. Wherever the checksum-and-transfer step runs, an artifact
written there is covered by it with no change to that step, so the artifact reaches long-term storage with the same
integrity guarantee as the data that produced it. A host configuring no storage destination runs neither the checksum
nor the transfer, and the artifact then stays local with the rest of the raw tree. The producer's verification runs
strictly before that step in the same function, so the artifact is present when the checksum is taken.

### Naming

One naming rule holds both seams, and it takes one of two shapes. When the tool's command line accepts an output name or
an output directory, the caller fixes it and both seams check that exact name. Only when the tool names its own output
does the rule degrade to a glob on a substring the tool's naming guarantees, plus a total order breaking ties. The
tie-break is required because a re-run leaves several matching files and discovery must stay deterministic.

Nothing validates at runtime that a file satisfying the producer's existence check also satisfies the consumer's rule,
so a tool configured under an unexpected project or output name ships successfully and is then invisible downstream.
Force the name where the tool's command line allows it, and record the rule as a module constant on each side whose
docstring names the other side.

### Shape

The artifact's format is read with a dependency the stack already carries, or with one added for this purpose. A format
readable only by importing the bound tool collapses the binding, because that import is the thing the blocking condition
forbids. A dependency added as a reader carries a comment naming the producing tool and the writer-side library version
on which the on-disk format depends, since that version governs whether the reader still parses the file.

The artifact's schema crosses the boundary alongside the argument vector's literals and the naming rule. The consumer
encodes that schema as constants, so a schema change at the tool is a change at the consumer, and no import warns of it.

---

## The two seams

Every binding has a producer seam, and it gains a consumer seam only when the platform reads the artifact back.

| Seam       | Where it lives                                       | What it owns                                                                   |
|------------|------------------------------------------------------|--------------------------------------------------------------------------------|
| `producer` | Acquisition-side preprocessing, sollertia-experiment | The gate, the vector, the log, the verification, the reap, and the naming rule |
| `consumer` | A processing stage, sollertia-forgery                | The locator, its registry donation, and the worker that reads the artifact     |

A binding's tool runs at the producer seam, where the session carries raw data alone. A tool needing parsed or
extracted input cannot bind there, because that input does not exist until processing has run, and it belongs in a
processing stage instead.

### The consumer seam is owned downstream

The consumer half is a module-level function donated to a per-acquisition-system registry, and
`forging:processing-input-format` owns its contract and the job discovery it gates. This skill states only
what the producer owes it, which is the directory named through the same session-record field, the naming rule recorded
as a constant on each seam, and the artifact's schema.

The registry receiving the donation may not exist. When a processing stage already reads this kind of artifact, the
donation joins that stage's registries. A new stage joins the pipeline whose input root already holds the artifact's
directory, and `forging:processing-input-format` tabulates those roots. Only an artifact under no existing input root
needs a new pipeline, which is the heaviest extension of the three. When none does, the consumer seam is a new stage.
That stage mints its own locator and worker registries, its pipeline call site, and its job-discovery and sizing
entries, all under `forging:library-extension`.

### An acquisition-only binding has no consumer seam

A binding whose artifact is terminal, shipped as raw data and never read by a processing stage, contributes no locator,
no worker, and no registry entry of its own. The producer half alone is a complete and conformant binding. The separate
obligation on the acquisition system to fill every registry the coverage check names, whether or not it binds anything,
belongs to `forging:processing-input-format`.

### The last seam is the dataset

A request ending at a forged dataset does not end at the worker. The worker writes a per-session table, and the dataset
columns that table feeds, along with the assembly routing filling them, are added under `forging:library-extension`. A
binding whose artifact must reach a dataset is complete only once that addition lands.

---

## Degradation

| Condition                                        | Seam       | Behavior                                                |
|--------------------------------------------------|------------|---------------------------------------------------------|
| The host is not configured for the tool          | `producer` | Return `None` silently, launch nothing                  |
| The session's own kind does not want the run     | `producer` | Skip the launch at the call site                        |
| The tool's input is missing before launch        | `producer` | Log at WARNING, return `None`, preprocessing continues  |
| The run fails, or exits zero writing no artifact | `producer` | Raise, before the session leaves the machine            |
| The pipeline aborts while a child still runs     | `producer` | Terminate, wait a bounded time, escalate to a kill      |
| The artifact is absent at read time              | `consumer` | Degrades again, under `forging:processing-input-format` |

A missing input and a failed run are two different paths, and a binding that merges them is wrong. A missing input means
the tool never ran, so the producer logs at WARNING, returns `None`, and preprocessing continues to completion. A tool
that ran and failed, or that exited zero having written no artifact, raises instead, which aborts the transfer to
long-term storage and retains the local session copy and the log tail for a manual retry. Both failure conditions are
checked together, because a tool exits zero having written nothing.

The child's streams are redirected to a file rather than a pipe, because a long-running child deadlocks on a full unread
pipe buffer. That file lives in the operating system's temporary directory under a name built from the session name,
never inside the session tree. A retained failure log is not raw data, so the checksum must not sweep it in. A
bounded tail of it is embedded in the failure message, and the unlink is placed after the raising call, so the log is
deleted on success and retained on failure. An asynchronous launch also terminates the child on abort, so an abandoned
process never becomes a second writer to the same output path on a retry.

**When a stage genuinely cannot proceed without the artifact**, do not make the consumer raise. Promote the requirement
upstream to the producer's verification, so a session reaching storage always carries the artifact, and let the
downstream readiness rules that `forging:processing-input-format` owns exclude a session that does not.

---

## What a binding is denied

| Denied                                                                      | Reason                                                                             |
|-----------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| An `AcquisitionSystems` member, or a sollertia-shared-assets registry entry | The platform dispatches on the system that runs the tool, never on the tool        |
| A pinned dependency in any stack member's `pyproject.toml`, and any import  | The environments are disjoint by construction, so the pin would not resolve        |
| A marketplace plugin under `plugins/`                                       | The binding's seams are documented by the plugins that own those seams             |
| An entry in the marketplace root README's library index                     | The index lists the stack, and a binding is listed in its own README section       |
| A sibling version pin cross-referenced against the stack                    | Stack members exact-pin one another, and a binding's version is chosen by the host |

The first row denies registry membership and indexing, not the session layout. An artifact landing in a session
directory that already exists needs no upstream change at all. An artifact needing its own directory adds a
`Directories` member and its session-record field under `assets:session-data`, which is the one sollertia-shared-assets
change a binding is permitted.

Three things cross the boundary, which are the argument vector's literals, the naming rule, and the artifact's on-disk
schema. None of them is an import, so a binding is verified by grep, since its distribution name appears in no
dependency list and its module name appears in no import.

Recorded instead, in these places:

| Where                                        | What it records                                                               |
|----------------------------------------------|-------------------------------------------------------------------------------|
| The marketplace root README bindings section | The tool, both seams, the blocking condition, and the runtime optionality     |
| The configuration section docstring          | Which fields gate the launch, and what an empty value means                   |
| The producer launcher docstring              | Why a subprocess, where the artifact lands, and that the checksum captures it |
| The consumer naming constant docstring       | The naming rule, and the producer that chooses the name                       |
| The bound repository's own packaging comment | The conflicting constraint, and the process boundary as its consequence       |

The first record is the `External tool bindings` section of the sollertia marketplace repository's root `README.md`,
which follows the indexed library categories and is the one place a binding is named for a reader rather than an agent.
The last two rows are conditional. The consumer naming constant docstring exists only for a binding the platform reads
back. The packaging comment exists only when the bound tool is a repository the platform authors, so a third-party or
operator-authored tool omits it and its blocking condition is carried by the launcher docstring and the README entry
alone.

---

## Workflow

You MUST follow these steps in order when adding a binding.

1. **Run the admission test.** Check the four blocking conditions and record the one that holds, with the evidence its
   shape allows. When none holds, stop and follow `assets:library-extension` instead.

2. **Check bindability.** Confirm the non-interactive entry point, the predictable output path, and the readable output
   format. A tool failing any of the three is not bound.

3. **Declare the configuration section.** Add a dataclass section to the acquisition system's configuration under
   `/acquisition-system-design`, with empty-defaulted identity fields for the tool's address and for its project,
   script, or model path, plus any parameter fields the run needs.

4. **Choose the artifact's home.** Reuse an existing raw-tree directory field when the artifact belongs beside the data
   that produced it. Otherwise add a `Directories` member in `data_hierarchy/session_data.py`, plus the matching field
   resolved in that dataclass's `build` classmethod. `assets:library-extension` owns that change and
   `assets:session-data` describes it.

5. **Fix the naming rule.** Pass the output name or directory explicitly where the tool's command line accepts one.
   Otherwise decide the substring and extension identifying the artifact, and the total order breaking ties. Record the
   rule as a module constant on each seam.

6. **Write the launcher.** Gate on the identity fields, and probe the input and return `None` with a WARNING when it is
   absent. Build the argument vector from the host-owned prefix and the tool's tokens, open the transient log file in
   the operating system's temporary directory under a session-derived name, and start the child.

7. **Verify the run.** Check the exit status and the artifact's existence together, raise with the log tail embedded,
   and unlink the log after the raising call. A synchronous launcher does this inline, and an asynchronous one does it
   in the caller's join. Place the verification before the checksum-and-transfer step that `/data-management` owns.

8. **Reap on abort, when the launch is asynchronous.** Wrap the stages running while the child is in flight, terminate
   the child on any exception, wait a bounded time, escalate to a kill, and re-raise.

9. **Decide whether the platform reads the artifact back.** When it does not, the producer half alone is the complete
   binding, so skip ahead to the recording step below.

10. **Write the consumer seam.** The locator and the worker are each a per-system donation, so each one joins the
    registry that already names its concern or mints a new one, under the "Minting or joining a per-system registry"
    scenario of `forging:library-extension`. When no stage reads this kind of artifact yet, mint the stage first through
    that skill's "Adding a new processing stage" recipe, then open the registry that stage needs. Then follow
    `forging:processing-input-format` for the locator's contract, and `forging:data-processing-design` for the worker
    itself.

11. **Add the reader dependency.** Declare the library that opens the artifact in the consuming package, with a comment
    naming the producing tool and the writer-side library version on which the format depends.

12. **Carry the artifact to the dataset, when the request ends there.** Add the dataset columns the worker's output
    feeds and the assembly routing that fills them, under `forging:library-extension` for the agnostic touches. The
    column itself and its emission site belong to the running system's schema skills, currently
    `mesoscope:mesoscope-vr-processing-schema` under "Adding an assembled column" and
    `mesoscope:mesoscope-vr-dataset-assembly` under "Emitting a new assembled column". The first of those also states
    the rebuild a new column forces on every dataset already defined.

13. **Record the binding.** Add the marketplace root README entry, and the packaging comment when the platform authors
    the bound repository. Then verify by grep that the tool's distribution name appears in no stack dependency list and
    its module name in no import.

---

## The worked instance

The platform carries one binding today, sollertia-video-tracking, whose `slvt` command runs DeepLabCut pose inference on
a face-camera video during Mesoscope-VR preprocessing. Its blocking conditions are a runtime mismatch and a dependency
conflict. DeepLabCut supports only Python 3.10 to 3.12 and the numpy 1.x series, so sollertia-video-tracking cannot
share the stack's environment. It targets the newest interpreter DeepLabCut supports, declaring `requires-python =
">=3.12,<3.13"` and `numpy>=1.26,<2`, both disjoint from the stack's `>=3.14,<3.15` and numpy 2. Read it as the one
instance and leave its values behind, because a second binding brings its own conflict, its own launcher, and its own
artifact format.

| Seam                     | Where it lives                                                        | Owning skill                            |
|--------------------------|-----------------------------------------------------------------------|-----------------------------------------|
| Configuration section    | `mesoscope_vr/system.py`, mounted on the system configuration         | `mesoscope:mesoscope-vr`                |
| Launcher, join, and reap | `mesoscope_vr/data_preprocessing.py`, around the preprocessing stages | `mesoscope:mesoscope-vr-runtime`        |
| Checksum and transfer    | `cross_system/data_preprocessing.py`, after the join                  | `/data-management`                      |
| Locator and worker       | `mesoscope_vr/video_tracking.py` in sollertia-forgery                 | `mesoscope:mesoscope-vr-video-tracking` |
| Registry donation        | `registries.py` in sollertia-forgery                                  | `forging:library-extension`             |

Three properties of this instance are choices rather than rules. It launches asynchronously because inference occupies
the rig's otherwise-idle GPU while the CPU- and disk-bound stages run, which a short synchronous tool would not need. It
addresses the tool by a configured environment name rather than by a host path, because the tool ships as a Python
distribution. Its call site gates the launch on the experiment session type, because only those sessions acquire the
data the predictions accompany.

Two cautions carry forward. Its producer checks existence by the input's stem while its consumer globs the tool's
project name, so one real filename satisfies both rules only while the operator names the tool's project as the consumer
expects. And neither seam names the other by repository or module, so the cross-reference this skill requires is a rule
the precedent does not yet meet.

---

## Related skills

| Skill                                      | Relationship                                                                             |
|--------------------------------------------|------------------------------------------------------------------------------------------|
| `assets:library-extension`                 | Owns the register path taken by a dependency the admission test admits                   |
| `assets:session-data`                      | Owns the session anatomy and the `Directories` member that names the artifact's home     |
| `/acquisition-system-design`               | Owns the configuration layer that hosts the binding's section                            |
| `/library-extension`                       | Owns the sollertia-experiment seams a new acquisition system composes                    |
| `/data-management`                         | Owns the checksum-and-transfer step and the preprocessing lifecycle                      |
| `forging:data-processing-design`           | Owns the worker pattern the consumer seam's reader follows                               |
| `forging:library-extension`                | Owns the registries, the new stage, and the dataset columns a consumer adds              |
| `forging:processing-input-format`          | Owns the donated locator's contract and the job discovery it gates                       |
| `mesoscope:mesoscope-vr`                   | Owns the worked instance's configuration section                                         |
| `mesoscope:mesoscope-vr-runtime`           | Owns the worked instance's launcher, join, abort reap, and options                       |
| `mesoscope:mesoscope-vr-video-tracking`    | Owns the worked instance's consumer stage                                                |
| `mesoscope:mesoscope-vr-processing-schema` | Owns the worked instance's assembled column, and the dataset rebuild a new column forces |
| `mesoscope:mesoscope-vr-dataset-assembly`  | Owns the worked instance's emission site that fills that column                          |

---

## Verification checklist

You MUST verify a binding against this checklist before reporting it complete.

```text
Binding compliance, tool-settled (run the greps named in each item):
- [ ] The tool's distribution name appears in no stack `pyproject.toml` dependency list
      (`grep -rn "<distribution-name>" */pyproject.toml`)
- [ ] The tool's module name appears in no stack import
      (`grep -rnE "^[[:space:]]*(import|from)[[:space:]]+<module>" */src/`)
- [ ] The tool appears in no `AcquisitionSystems` member and no sollertia-shared-assets registry
- [ ] No `plugins/` directory carries a plugin for the bound tool
- [ ] The marketplace root README library index carries no entry, and its bindings section carries one

Binding compliance, reader-judged:
- [ ] The blocking condition is named, with the evidence its shape allows
- [ ] All three bindability conditions hold
- [ ] Every identity field defaults to an empty value, and the launcher launches only when all of them are set
- [ ] No filesystem path, environment name, or installation root for the tool appears as a code constant
- [ ] The argument vector is a `list[str]` with no shell, every value stringified at the call site
- [ ] An empty parameter value omits its argument rather than passing an empty one
- [ ] A synchronous launcher verifies inline, and an asynchronous one returns a handle or `None` and is joined
- [ ] The artifact's directory is named through a session-record field on both seams
- [ ] The artifact lands inside the session tree, in the raw tree unless a reason is recorded, and the verification
      runs before the checksum-and-transfer step
- [ ] The naming rule is a forced exact name, or a glob plus a total order where the tool names its own output
- [ ] The producer's existence check and the consumer's discovery are satisfied by that one naming rule
- [ ] The artifact's format is read without importing the bound tool
- [ ] A missing input logs at WARNING and returns `None`, and preprocessing continues to completion
- [ ] A failed run and a zero exit writing no artifact are checked together, and both raise before the transfer
- [ ] The child's streams go to a file in the temporary directory, and the unlink follows the raising call
- [ ] An asynchronous launch terminates the child on abort with a bounded wait escalating to a kill
- [ ] The consumer donation joins the registry naming its concern or mints one, with a new stage minted first
- [ ] An artifact that must reach a forged dataset carries its dataset columns and assembly routing
- [ ] The reader dependency comment names the tool and the writer-side version on which the format depends
- [ ] Every recording place that applies to this binding carries it
```
