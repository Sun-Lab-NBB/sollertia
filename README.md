# Sollertia

**A platform for AI-assisted scientific data acquisition and processing**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

Sollertia is an open-source platform for running behavioral neuroscience experiments end to end, from designing an
acquisition system to forging the recorded sessions into analysis-ready datasets. It builds on the
[Ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) framework, which supplies the hardware interface libraries and the
timing primitives underneath. Above them, Sollertia adds the layers a laboratory needs: a shared asset vocabulary, an
acquisition runtime, a Virtual Reality task engine, and a processing pipeline.

The platform separates what every acquisition system shares from what one system contributes. System-agnostic libraries
own the record schemas, the batch orchestration, and the extension seams, while a per-system package donates the
parsers, locators, and workers that make those seams concrete. Mesoscope-VR is the reference system, and it is the
worked example an agent copies when implementing a new one.

**Core Insight:** An agent sets the work up and then gets out of the way. In acquisition, it configures the system and
leaves before an animal reaches the rig. In processing, it plans and deploys the batches, then monitors what a
*deterministic internal orchestrator* resolves. A dropped network connection, an API rate limit, or a model error never
reaches a running session or a running batch.

Authored by [Ivan Kondratyev](https://github.com/Inkaros).
Copyright: 2026, NeuroAI Lab, Cornell University.

___

## Features

### Shared Asset Vocabulary
- **One record schema per concern**: Session markers, descriptors, hardware-state snapshots, and experiment
  configurations are defined once and dispatched to every system through registries.
- **Registry-backed extension**: A new acquisition system or session type gains its enumeration member in one place, and
  import-time coverage checks refuse a partially wired system before runtime begins.
- **Validated authoring**: Every asset is created, written, and validated through MCP tools that check the payload
  before it reaches disk.

### Acquisition Runtime
- **System-agnostic orchestration**: A lifecycle orchestrator owns master start and stop, per-mode logic functions, and
  the state machine every acquisition system instantiates.
- **Hardware binding classes**: Microcontrollers, cameras, and Zaber motors are composed into per-subsystem bindings
  driven from one system configuration file.
- **Virtual Reality coupling**: A task driver bridges the Python runtime and a Unity game engine over MQTT, decomposing
  wall-cue sequences into trials as the session runs.

### Data Processing
- **Six batch pipelines**: Checksum, runtime, microcontroller, video, two-photon, and dataset forging, each planned,
  prepared, and executed through one generic job model.
- **Local or scheduler execution**: The same batch runs on a local process pool or as one SLURM allocation per job on a
  configured compute server.
- **Auditable job state**: Every job carries a tracked status on a per-unit tracker, so a rerun resolves only the work
  still outstanding.

### AI-Assisted Development
- **Sixty-six skills across five plugins**: Encoded conventions, record schemas, extension recipes, and end-to-end
  pipeline orchestration.
- **Three MCP servers**: Structured tool access to shared assets, the acquisition system, and the processing pipeline.
- **Deliberate handoffs**: Each skill declares what it owns and what it defers, so an agent reaches the one skill that
  answers its question.

___

## Architecture

```text
                    ataraxis  (hardware interfaces, timing, logging, transport)
                                         |
                    sollertia-shared-assets  (record schemas, registries, slsa)
                            /                |                    \
        sollertia-experiment          sollertia-virtual-reality    sollertia-forgery
        (acquisition, sle)            (Unity VR tasks)             (processing, slf)
                |                              |                          |
    sollertia-micro-controllers        MQTT task contract          cindra (imaging)
    (AXMC firmware)                                                       |
                                                                          |
                                                                    forged datasets
```

Shared assets sit at the center, because both the acquisition side and the processing side read the same records. The
acquisition libraries write a session, and the processing library reads it back without either side importing the
other. Neither side ever imports a per-system package directly, since every system-specific asset resolves through a
registry keyed by the acquisition system recorded in the session itself.

___

## Design Invariants

The platform declines to generalize in two places, and it does so deliberately. The registries exist to make
everything around those two boundaries mechanical, so the boundaries themselves stay small, explicit, and stated in
the skills an agent reaches on the way to them.

**The acquisition system engine is a human-in-the-loop rewrite.** An engine is defined by a hardware inventory and a
set of laboratory-local wiring conventions that no template can carry without misrepresenting them, so the platform
exposes no runtime base class to subclass. It supplies two scaffolds instead: the Mesoscope-VR package, from which a new
engine is copied, and the domain-agnostic primitives in `sollertia-experiment`'s `cross_system` package, which a new
engine composes. An agent scaffolds the engine and owns every registry seam that glues it into the rest of the
platform. The hardware inventory, the wiring topology, the per-mode semantics of the state machine, the calibration
values, the safety interlocks, and the teardown ordering are settled with the human supervisor who owns the rig.

**Virtual Reality is the linear infinite corridor.** Every acquisition system presents a Unity task in the linear
infinite corridor, as the `AcquisitionSystems` enumeration states in its own docstring, so every experiment
configuration is seeded from a corridor task template and satisfies one contract. Authoring and validating new corridor
templates is autonomous work. A different topology, such as a T-maze, an open field, or a branching maze, is a second
task engine, and it is co-designed with the human supervisor.

Neither boundary is a missing feature. Every other extension the platform supports, including session types, hardware
modules, processing pipelines, forged datasets, read assets, and credentials, resolves through a registry. The
shared-assets and forgery registries run import-time coverage checks that refuse a partially wired extension before a
session reaches disk.

___

## Libraries

### Shared Foundation

- **[sollertia-shared-assets](https://github.com/Sun-Lab-NBB/sollertia-shared-assets)** (Python). Defines every record
  the platform reads and writes, the registries that dispatch them per acquisition system, and the `slsa` MCP server
  that authors and validates them. Both sollertia-experiment and sollertia-forgery depend on it.

### Data Acquisition

- **[sollertia-experiment](https://github.com/Sun-Lab-NBB/sollertia-experiment)** (Python). The acquisition runtime,
  which composes hardware binding classes into an acquisition system, runs the session state machine, and preprocesses
  the recorded data for transfer. Ships the `sle` CLI and MCP server.
- **[sollertia-micro-controllers](https://github.com/Sun-Lab-NBB/sollertia-micro-controllers)** (C++). The firmware
  running on every Ataraxis Micro Controller the platform drives, one module class per hardware device.
- **[sollertia-virtual-reality](https://github.com/Sun-Lab-NBB/sollertia-virtual-reality)** (C#, Unity). The Virtual
  Reality task engine. Generates corridor tasks from YAML templates and exchanges runtime state with the acquisition
  runtime over MQTT.

### Data Processing

- **[sollertia-forgery](https://github.com/Sun-Lab-NBB/sollertia-forgery)** (Python). Turns recorded sessions into
  per-session data tables and multi-session datasets across six batch pipelines, locally or on a SLURM cluster. Ships
  the `slf` CLI and MCP server.

___

## External Tool Bindings

Some tools the platform runs cannot live inside it. An external tool binding is reached across a process boundary,
because its runtime, its dependency pins, its license, or its own launcher forbids installing or driving it beside the
stack. The binding's contract is the artifact it leaves on disk. A binding is not indexed above, ships no marketplace
plugin, and is not version-checked as a sibling clone. It is documented at the two seams it touches, the call that
produces the artifact and the stage that reads it back. The `experiment:external-tool-bindings` skill owns the
convention and the producer seam, and `forging:processing-input-format` owns the consumer seam.

- **[sollertia-video-tracking](https://github.com/Sun-Lab-NBB/sollertia-video-tracking)** (Python, DeepLabCut). Injected
  into Mesoscope-VR acquisition preprocessing, which invokes its `slvt infer` command through `conda run` and leaves the
  DeepLabCut prediction beside the face-camera video. The forging video pipeline reads that prediction back to compute
  pupil metrics. DeepLabCut supports only Python 3.10 to 3.12 and the numpy 1.x series, so it cannot share the stack's
  Python 3.14 and numpy 2 environment, and the project pins itself to the newest interpreter DeepLabCut allows. The
  binding is skipped where the host leaves it unconfigured or the face-camera video is absent, while an inference that
  runs and fails aborts the session transfer.

___

## Getting Started

### Installation

The libraries do not share one distribution channel, so install each from the channel that publishes it.

Python libraries are available via PyPI and require Python `>=3.14,<3.15`:

```bash
pip install sollertia-shared-assets sollertia-experiment sollertia-forgery
```

An external tool binding installs into its own environment instead, on the interpreter its own pins allow. For
sollertia-video-tracking that is Python `>=3.12,<3.13`, and the acquisition configuration names that environment.

C++ firmware is built and uploaded with PlatformIO from a clone of the repository, and the Unity project is opened
directly in the Unity Editor. Both carry their own setup instructions in their READMEs.

___

## Claude Code Plugins

This repository serves as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin marketplace. It
distributes five plugins carrying sixty-six skills that encode the platform's record schemas, extension recipes, and
pipeline workflows, along with the MCP servers that expose the libraries to AI agents. Installing a plugin makes its
skills available to Claude Code and, for the plugins that bundle an MCP server, registers that server automatically.

| Plugin       | MCP Server                | Skills | Focus                                                                    |
|--------------|---------------------------|--------|--------------------------------------------------------------------------|
| `assets`     | `sollertia-shared-assets` | 13     | Record schemas, session discovery, dataset inspection, the Unity relay.  |
| `experiment` | `sollertia-experiment`    | 15     | Acquisition-system design, hardware interfaces, runtime, and extension.  |
| `forging`    | `sollertia-forgery`       | 14     | Batch processing, job planning, dataset forging, and remote execution.   |
| `mesoscope`  | —                         | 13     | The Mesoscope-VR reference system, acquisition side and processing side. |
| `unity`      | —                         | 11     | Unity Editor integration, VR task generation, scenes, and Play Mode.     |

The `mesoscope` and `unity` plugins register no server of their own. `mesoscope` requires the `assets`, `experiment`,
and `forging` plugins, and `unity` reaches the Unity Editor relay through the `assets` plugin's server.

### Installation

Claude Code installs plugins through its built-in marketplace system. First, add the sollertia marketplace:

`/plugin marketplace add Sun-Lab-NBB/sollertia`

Then install any combination of the five plugins:

`/plugin install assets@sollertia`

`/plugin install experiment@sollertia`

`/plugin install forging@sollertia`

`/plugin install mesoscope@sollertia`

`/plugin install unity@sollertia`

Alternatively, run `/plugin`, open the **Discover** tab, and select the plugins to install interactively.

***Note,*** installing the `assets`, `experiment`, or `forging` plugin automatically registers its bundled MCP server,
started via `slsa mcp`, `sle mcp`, and `slf mcp` respectively. No manual edits to `~/.claude.json` are required. When a
plugin is enabled mid-session, run `/reload-plugins` to connect its MCP server. The matching pip package must also be
installed in the Python environment active when Claude Code starts, since the server launches through the `slsa`,
`sle`, or `slf` command and fails to start without it.

Most Sollertia skills are deliberately **not** user-invocable. They encode background conventions and domain knowledge
that AI coding agents pick up automatically whenever a task matches the skill's description, so there is nothing to
type. The one exception performs a discrete on-demand action and can be typed as a slash command.

| Command                | Description                                                                        |
|------------------------|------------------------------------------------------------------------------------|
| `/system-health-check` | Verifies platform configuration, network mounts, hardware, and configuration files |

Each plugin also depends on the ataraxis and cindra marketplaces. Add both, then install `automation` for the
development conventions, `video`, `communication`, and `microcontroller` for the hardware and log-processing skills that
receive the sollertia skills' handoffs, and `cindra` for the imaging stages the forging pipelines dispatch:

`/plugin marketplace add Sun-Lab-NBB/ataraxis`

`/plugin marketplace add Sun-Lab-NBB/cindra`

`/plugin install automation@ataraxis`

`/plugin install video@ataraxis`

`/plugin install communication@ataraxis`

`/plugin install microcontroller@ataraxis`

`/plugin install cindra@cindra`

### assets

| Skill                          | Description                                                         |
|--------------------------------|---------------------------------------------------------------------|
| `working-directory`            | Initializes the local working directory, data root, and credentials |
| `project-hierarchy`            | Discovers the project hierarchy and creates and removes projects    |
| `session-discovery`            | Discovers sessions under a project root and filters the result      |
| `session-data`                 | Reads, writes, and validates SessionData markers and health reports |
| `session-descriptors`          | Reads, writes, and validates per-session-type descriptor YAMLs      |
| `session-hardware-state`       | Reads, writes, and validates per-system hardware-state YAMLs        |
| `experiment-configuration`     | Authors per-project, per-system experiment configuration YAMLs      |
| `task-templates`               | Authors, modifies, and validates reusable task template YAMLs       |
| `data-assets`                  | Reads, writes, and describes on-disk read-asset dataclasses         |
| `datasets`                     | Discovers, inspects, reads, writes, and validates forged datasets   |
| `library-extension`            | Owns the extension path of the shared-assets registry system        |
| `cli-reference`                | Documents the human-facing `slsa` command-line interface            |
| `assets-mcp-environment-setup` | Diagnoses and resolves MCP server connectivity issues               |

### experiment

| Skill                              | Description                                                               |
|------------------------------------|---------------------------------------------------------------------------|
| `system-design-pipeline`           | Orders the phases of designing and building a new acquisition system      |
| `pipeline`                         | Orders the phases of the experiment lifecycle from bringup to handoff     |
| `acquisition-system-design`        | Documents the configuration and binding-class layer of a system           |
| `acquisition-system-runtime`       | Documents the runtime state machine and per-mode logic layer              |
| `acquisition-system-setup`         | Discovers and verifies the hardware connected to an acquisition PC        |
| `microcontroller-interface`        | Registry of the paired Module and ModuleInterface classes                 |
| `zaber-interface`                  | Guides implementation of Zaber motor interfaces using zaber-motion        |
| `vr-driver-interface`              | Documents the Virtual Reality task driver subsystem and its MQTT contract |
| `google-sheets-processing`         | Guides implementation of the SurgeryLog and WaterLog sheet processors     |
| `data-management`                  | Preprocesses, migrates, and deletes session data after acquisition        |
| `external-tool-bindings`           | Owns the external tool binding convention and its producer seam           |
| `library-extension`                | Owns the extension path of the acquisition runtime and the firmware       |
| `system-health-check`              | Verifies platform configuration, mounts, hardware, and configurations     |
| `cli-reference`                    | Documents the system-agnostic half of the `sle` command-line interface    |
| `experiment-mcp-environment-setup` | Diagnoses and resolves MCP server connectivity issues                     |

### forging

| Skill                           | Description                                                                |
|---------------------------------|----------------------------------------------------------------------------|
| `pipeline`                      | Orders the phases of the processing lifecycle from setup to project state  |
| `job-planning`                  | Sizes every runnable job of a session or a dataset                         |
| `batch-processing`              | Orchestrates batch preparation, job execution, monitoring, and cancelation |
| `dataset-definition`            | Composes and grows forged dataset hierarchies and reports their state      |
| `dataset-forging`               | Documents what the forging pipeline does differently from the other five   |
| `remote-execution`              | Runs processing work on the configured SLURM compute server                |
| `server-configuration`          | Authors the ServerConfiguration YAML that authorizes remote execution      |
| `processing-input-format`       | Documents the on-disk inputs each batch pipeline requires                  |
| `processing-results`            | Documents what each pipeline writes to disk and how to verify it           |
| `project-state`                 | Documents the session manifest and the job table published beside it       |
| `data-processing-design`        | Documents the agnostic worker and per-system donation design pattern       |
| `library-extension`             | Owns the extension path of the processing library and its registries       |
| `cli-reference`                 | Documents the human-facing `slf` command-line interface                    |
| `forging-mcp-environment-setup` | Diagnoses and resolves MCP server connectivity issues                      |

### mesoscope

| Skill                                 | Description                                                            |
|---------------------------------------|------------------------------------------------------------------------|
| `mesoscope-vr`                        | Documents the Mesoscope-VR hardware inventory and configuration layer  |
| `mesoscope-vr-runtime`                | Documents the state machine, the orchestrator, and the control GUIs    |
| `mesoscope-vr-session-schema`         | Documents the four session descriptors and the hardware-state snapshot |
| `mesoscope-vr-experiment-schema`      | Documents the experiment configuration and trial class schema          |
| `mesoscope-vr-snapshots`              | Reads and writes the per-session frozen position snapshots             |
| `mesoscope-vr-module-parsing`         | Documents the eight-entry microcontroller module parser registry       |
| `mesoscope-vr-trial-decomposition`    | Documents the runtime-log cue-sequence to trial decomposition          |
| `mesoscope-vr-fluorescence-alignment` | Documents the fluorescence frame alignment sub-assembly                |
| `mesoscope-vr-video-tracking`         | Documents the pupil-tracking pass and video sub-dataset assembler      |
| `mesoscope-vr-imaging-configuration`  | Documents the two-photon donations and cindra configuration resolvers  |
| `mesoscope-vr-dataset-assembly`       | Documents the session-assembly worker and its admission policy         |
| `mesoscope-vr-processing-schema`      | Documents the filename rosters and the assembled column roster         |
| `mesoscope-vr-cli-reference`          | Documents the human-facing `sle mesoscope` command group               |

### unity

| Skill                         | Description                                                            |
|-------------------------------|------------------------------------------------------------------------|
| `task-prefabs`                | Creates, deletes, and inspects Unity tasks from YAML task templates    |
| `task-scenes`                 | Manages task scenes and project asset enumeration                      |
| `task-parameters`             | Reads and writes the consolidated Task Parameters editor window        |
| `play-mode`                   | Controls Unity Editor Play Mode through the relay                      |
| `zone-prefabs`                | Manufactures new trigger zone prefabs from the canonical base prefabs  |
| `scene-setup`                 | Guides Editor-side scene configuration ahead of Play Mode              |
| `task-generator`              | Documents the `CreateTask.cs` editor pipeline that builds task prefabs |
| `gimbl-framework`             | Reference for the inlined GIMBL VR framework components                |
| `mqtt-contract`               | Documents the bidirectional MQTT topic contract with the runtime       |
| `unity-tests`                 | Documents the Unity Test Framework suite and the assembly layout       |
| `unity-mcp-environment-setup` | Diagnoses and resolves Unity Editor relay connectivity issues          |

___

## Example Workflows

### Designing a New Acquisition System

```text
> I want to build a new acquisition system for a head-fixed treadmill rig with two cameras.

The agent invokes experiment:system-design-pipeline, which orders the build across four repositories.
It then walks the phases: registering the AcquisitionSystems member and session types through
assets:library-extension, composing the binding classes through experiment:acquisition-system-design,
and wiring the processing donations through forging:library-extension. At each step it reads the
Mesoscope-VR instance as the worked example.
```

### Processing a Week of Sessions

```text
> Process every session recorded for project Alpha last week and forge them into a dataset.

The agent invokes forging:pipeline, which orders the run. It discovers the sessions through
assets:session-discovery, sizes the work through forging:job-planning, then prepares and executes each
batch through forging:batch-processing. It reads the outcome through forging:project-state, composes the
dataset through forging:dataset-definition, and forges it.
```

___

## Adoption Roadmap

1. **Install the marketplaces.** Add `sollertia`, `ataraxis`, and `cindra`, then install the plugins matching the
   adopter's side of the platform. Acquisition needs `assets` and `experiment`, processing needs `assets` and `forging`.
2. **Bootstrap the working directory.** Set the local working directory, the data root, and the platform credentials,
   against which every other tool resolves its paths.
3. **Bring up the hardware.** Discover the connected cameras, microcontrollers, and motors, and reconcile what is found
   against the system configuration file.
4. **Author the task and the experiment.** Design a Virtual Reality task template, generate its Unity task, and author
   the experiment configuration that instantiates it.
5. **Record and process a session.** Run one acquisition session, preprocess it, then plan and execute the processing
   pipelines against it before scaling to a project.
6. **Implement a new system.** Mesoscope-VR is the reference instance, and the `mesoscope` plugin documents every
   donation it makes. Scaffold the new engine from its shape, and expect to settle the hardware-defined decisions with
   the human supervisor while an agent wires the registry seams around them.

___

## License

Every repository indexed above is released under the Apache License 2.0. See [LICENSE](LICENSE) for the full text.
The cindra imaging library, which the architecture diagram shows beneath sollertia-forgery and which that library
declares as a runtime dependency, is released under the GNU General Public License v3.0 or later instead.

___

## Acknowledgments

This platform builds on the [Ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) framework, on
[DeepLabCut](https://github.com/DeepLabCut/DeepLabCut) for pose estimation, and on [Unity](https://unity.com/) for the
Virtual Reality task engine.

Developed in the Sun (NeuroAI) lab at Cornell University. Questions and contributions are welcome through the issue
tracker of the repository the question concerns.
