# Sollertia

**AI-Assisted Scientific Data Acquisition and Processing**

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

Sollertia is an open-source platform for running behavioral neuroscience experiments end to end, from designing an
acquisition system to forging the recorded sessions into analysis-ready datasets. It builds on the
[Ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) framework, which supplies the hardware interface libraries and the
timing primitives underneath, and adds the layers a laboratory needs above them: a shared asset vocabulary, an
acquisition runtime, a Virtual Reality task engine, and a processing pipeline.

The platform separates what every acquisition system shares from what one system contributes. System-agnostic libraries
own the record schemas, the batch orchestration, and the extension seams, while a per-system package donates the
parsers, locators, and workers that make those seams concrete. Mesoscope-VR is the reference system, and it is the
worked example an agent copies when implementing a new one.

**Core Insight:** AI assistance operates at *configuration time* and at *processing time*, while runtime data
acquisition remains *deterministic and AI-independent*. An agent designs the system, authors its configurations, and
processes its output, and no agent sits in the loop while an animal is on the rig.

Authored by [Ivan Kondratyev](https://github.com/Inkaros).
Copyright: 2026, NeuroAI Lab, Cornell University.

___

## Features

### Shared Asset Vocabulary
- **One record schema per concern**: Session markers, descriptors, hardware-state snapshots, and experiment
  configurations are defined once and dispatched to every system through registries.
- **Registry-backed extension**: A new acquisition system or session type gains its enumeration member in one place,
  and import-time coverage checks refuse a partially wired system rather than failing at runtime.
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
- **Sixty-five skills across five plugins**: Encoded conventions, record schemas, extension recipes, and end-to-end
  pipeline orchestration.
- **Three MCP servers**: Structured tool access to shared assets, the acquisition system, and the processing pipeline.
- **Deliberate handoffs**: Each skill declares what it owns and what it defers, so an agent reaches the one skill that
  answers its question rather than guessing.

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

## Libraries

### Shared Foundation

- **[sollertia-shared-assets](https://github.com/Sun-Lab-NBB/sollertia-shared-assets)** (Python). Defines every record
  the platform reads and writes, the registries that dispatch them per acquisition system, and the `slsa` MCP server
  that authors and validates them. Every other Python library depends on it.

### Data Acquisition

- **[sollertia-experiment](https://github.com/Sun-Lab-NBB/sollertia-experiment)** (Python). The acquisition runtime.
  Composes hardware binding classes into an acquisition system, runs the session state machine, and preprocesses the
  recorded data for transfer. Ships the `sle` CLI and MCP server.
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
stack, and its contract is the artifact it leaves on disk rather than an API. A binding is not indexed above, ships no
marketplace plugin, and is not version-checked as a sibling clone. It is documented at the two seams it touches, the
call that produces the artifact and the stage that reads it back. The `experiment:external-tool-bindings` skill owns
the convention and the producer seam, and `forging:processing-input-format` owns the consumer seam.

- **[sollertia-video-tracking](https://github.com/Sun-Lab-NBB/sollertia-video-tracking)** (Python, DeepLabCut). Bolted
  into Mesoscope-VR acquisition preprocessing, which invokes its `slvt infer` command through `conda run` and leaves
  the DeepLabCut prediction beside the face-camera video. The forging video pipeline reads that prediction back to
  compute pupil metrics. DeepLabCut supports only Python 3.10 to 3.12 and the numpy 1.x series, so it cannot share the
  stack's Python 3.14 and numpy 2 environment, and the project pins itself to the newest interpreter DeepLabCut
  allows. The binding is skipped where the host leaves it unconfigured or the face-camera video is absent, while an
  inference that runs and fails aborts the session transfer rather than passing silently.

___

## Getting Started

### Installation

The libraries do not share one distribution channel, so install each from the channel that publishes it.

Python libraries are available via PyPI and require Python `>=3.14,<3.15`:

```bash
pip install sollertia-shared-assets sollertia-experiment sollertia-forgery
```

An external tool binding installs into its own environment instead, on the interpreter its own pins allow. For
sollertia-video-tracking that is Python `>=3.12,<3.13`, and the acquisition configuration names the environment it
created.

C++ firmware is built and uploaded with PlatformIO from a clone of the repository, and the Unity project is opened
directly in the Unity Editor. Both carry their own setup instructions in their READMEs.

___

## Claude Code Plugins

This repository serves as a [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin marketplace. It
distributes five plugins carrying sixty-five skills that encode the platform's record schemas, extension recipes, and
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

Each plugin also depends on the ataraxis marketplace for development-convention skills, so add that marketplace and
install its `automation` plugin alongside these:

`/plugin marketplace add Sun-Lab-NBB/ataraxis`

`/plugin install automation@ataraxis`

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

1. **Install the marketplaces.** Add both `sollertia` and `ataraxis`, then install the plugins matching the side of the
   platform you work on. Acquisition needs `assets` and `experiment`, processing needs `assets` and `forging`.
2. **Bootstrap the working directory.** Set the local working directory, the data root, and the platform credentials,
   which every other tool resolves its paths against.
3. **Bring up the hardware.** Discover the connected cameras, microcontrollers, and motors, and reconcile what is found
   against the system configuration file.
4. **Author the task and the experiment.** Design a Virtual Reality task template, generate its Unity task, and author
   the experiment configuration that instantiates it.
5. **Record and process a session.** Run one acquisition session, preprocess it, then plan and execute the processing
   pipelines against it before scaling to a project.
6. **Implement your own system.** Mesoscope-VR is the reference instance, and the `mesoscope` plugin documents every
   donation it makes. Copy its shape rather than inventing a new one.

___

## License

Every repository indexed above is released under the Apache License 2.0. See [LICENSE](LICENSE) for the full text.

___

## Acknowledgments

This platform builds on the [Ataraxis](https://github.com/Sun-Lab-NBB/ataraxis) framework, on
[DeepLabCut](https://github.com/DeepLabCut/DeepLabCut) for pose estimation, and on
[Unity](https://unity.com/) for the Virtual Reality task engine.

Developed in the Sun (NeuroAI) lab at Cornell University. Questions and contributions are welcome through the issue
tracker of the repository the question concerns.
