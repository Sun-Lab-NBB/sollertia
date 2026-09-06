---
name: cli-reference
description: >-
  Documents the system-agnostic half of the sle command-line interface of the sollertia-experiment library. Covers the
  root group, the mcp command, and the six sle get commands with every option's short form, long form, type, default,
  and effect. Also covers the MCP tool to which each command maps and how the CLI path diverges from the MCP path. Use
  when a user asks what an sle, sle mcp, or sle get command or option does, or when the MCP server is unavailable and
  the user must be told what to run by hand.
user-invocable: false
---

# CLI reference

> **The `sle` CLI is a HUMAN-FACING tool. You MUST never invoke it.** Print the command, ask the user to run it, and
> ask them to paste the output back.

**The one exemption** is `--help`. `sle --help` and `sle COMMAND --help` may be run, and no other `sle` invocation is
exempt. Every `sle get` command except `ports` and `checksum` opens the host's serial ports, cameras, or the Unity
bridge, which is experimenter territory. The `ports` command only enumerates port names and `checksum` touches no device
at all, but the CLI is still the experimenter's to run. `/experiment-mcp-environment-setup` owns the `sle --help` smoke
test.

---

## Scope

**Covers:**
- The complete system-agnostic `sle` command surface: every Click node, its purpose, and the MCP tool to which it maps
- Every declared option: short form, long form, type, default, required, flag, or prompted status, and effect
- Per-command output shapes, empty-result behavior, and the exception each failure raises
- Which CLI commands have no MCP equivalent, which agnostic MCP tools have no CLI equivalent, and where pairs differ
- The pattern by which a registered acquisition system contributes its own `sle <system>` command group
- What to tell a user to run when the MCP server cannot be restored in this session

**Does not cover:**
- Every `sle mesoscope` command and option. Owned by `mesoscope:mesoscope-vr-cli-reference`.
- Diagnosing why the MCP server is down. Owned by `/experiment-mcp-environment-setup`.
- The hardware discovery workflow and how to read its output. Owned by `/acquisition-system-setup`.
- Zaber motor semantics, configuration reads, and configuration writes. Owned by `/zaber-interface`.
- The Unity bridge. Owned by `/vr-driver-interface` for the driver side and `unity:unity-mcp-environment-setup` for
  the editor-side relay.

**Handoff rules:** If the user wants an operation performed rather than explained, use the MCP tools and invoke the
owning skill. If the MCP tools are unavailable, invoke `/experiment-mcp-environment-setup` first, and fall back to the
handoff table below only after the server cannot be restored. If the question names a command group that is neither
`mcp` nor `get`, that group belongs to an acquisition system, so route it as the extension point section directs.

---

## Agent requirements

You MUST answer CLI questions from this skill or from `sle COMMAND --help`, never from memory. When a user's report
disagrees with this reference, ask them to run `sle COMMAND --help` and read the installed build's answer.

You MUST NOT infer an MCP tool name from a command name. Three of the six `sle get` commands have no tool on this
server at all, and four agnostic tools have no command, so the two surfaces are read separately rather than mapped.

The agnostic `sle` surface addresses this machine only.

---

## Command surface

The system-agnostic surface declares nine Click nodes: the root group, the `mcp` command, the `get` group, and six
`get` leaf commands. The entry point is `sle = "sollertia_experiment.interfaces.entry_points:sle_cli"`, declared in the
`[project.scripts]` table of `pyproject.toml`. Each registered acquisition system attaches one further group to the
root, which the count above excludes and the extension point section below routes.

| Click node            | Kind    | Purpose                                                           | MCP equivalent                   |
|-----------------------|---------|-------------------------------------------------------------------|----------------------------------|
| `sle`                 | group   | Entry point. Dispatches to the subcommands and prints the listing | None, dispatch only              |
| `sle mcp`             | command | Starts the MCP server over the named transport                    | None, this command IS the server |
| `sle get`             | group   | Dispatches the six system-agnostic discovery commands             | None, dispatch only              |
| `sle get zaber`       | command | Prints the port, device, and axis table of every Zaber device     | `get_zaber_devices_tool`         |
| `sle get cameras`     | command | Prints every accessible camera, split by interface                | None, `axvs` owns the tool       |
| `sle get controllers` | command | Probes every serial port for a microcontroller identifier         | None, `axci` owns the tool       |
| `sle get ports`       | command | Prints every serial port carrying a USB product identifier        | None on any server               |
| `sle get unity`       | command | Reports whether the Unity Editor MCP bridge answers               | `check_unity_bridge_tool`        |
| `sle get checksum`    | command | Prints the CRC32-XFER checksum of the supplied string             | `get_checksum_tool`              |

**Note on `-h`:** `_CONTEXT_SETTINGS` sets `max_content_width` alone, so Click's `help_option_names` stays at its
`["--help"]` default and `-h` is never a help alias. No agnostic node binds `-h` to anything either, so `sle get -h`
aborts on an unknown option at exit code 2.

### The per-system extension point

`_register_subcommands()` in `interfaces/entry_points.py` imports the `get` group and one group per registered
acquisition system, then attaches each through `sle_cli.add_command()`. The helper runs at module import, so every
registered group is present on every `sle` invocation, including `--help`. The helper carries no discovery mechanism,
and its imports are hardcoded, so a new system's group arrives through an edit to it. `/library-extension` owns that
seam.

The reading rule follows from the shape rather than from any catalogue. A `sle` command whose first segment is neither
`mcp` nor `get` names an acquisition system, and that system's own plugin documents its commands and options. You MUST
route such a question to the owning plugin rather than answering it here, because nothing about a system group is
derivable from this module.

`AcquisitionSystems` holds one member today, so `sle mesoscope` is currently the only such group.
`mesoscope:mesoscope-vr-cli-reference` documents every one of its commands and every one of their options.

---

## Option reference

The whole agnostic surface declares exactly two options, one on `sle mcp` and one on `sle get checksum`. Both are
declared in `interfaces/entry_points.py` and `interfaces/get.py`. Every other node declares none and accepts only
`--help`. The root group, the `mcp` command, and the `get` group each set `max_content_width` to 120 explicitly, and
every `get` leaf inherits its parent's context, so every `--help` rendering wraps at that width.

### `sle` and `sle get`

Neither group declares an option. A bare `sle` or a bare `sle get` prints the group listing to stderr and exits 2,
because Click 8.4.2 raises `NoArgsIsHelpError`, a `UsageError` subclass, for a group invoked with no subcommand.
Neither group parses a shared option on behalf of its children, so no ordering rule applies and an option may be
written wherever its own command accepts it.

### `sle mcp`

| Short | Long          | Type     | Default   | Form     | Effect                                             |
|-------|---------------|----------|-----------|----------|----------------------------------------------------|
| `-t`  | `--transport` | `Choice` | `"stdio"` | optional | Transport. Choice of `stdio` and `streamable-http` |

The choice is case-insensitive and `show_default` is set, so the rendered help names `stdio`. `stdio` calls
`console.disable()`, because one echoed status line inside the JSON-RPC stream the two share makes a message
unparsable. `streamable-http` keeps the console on and echoes one startup line before it blocks.
`/experiment-mcp-environment-setup` owns the transport branches and the startup line as a diagnostic.

### `sle get checksum`

| Short | Long             | Type  | Default        | Form     | Effect                                                |
|-------|------------------|-------|----------------|----------|-------------------------------------------------------|
| `-i`  | `--input-string` | `str` | none, prompted | optional | The string to checksum. Omitted, Click prompts for it |

The option declares no `type`, so Click applies its `STRING` default, and it declares no `default`, so the `prompt`
argument supplies the value interactively when the option is absent. The declared prompt text already ends in a colon
and a space, and Click appends its own `": "` suffix, so the rendered prompt carries both. A prompted invocation is
interactive, so a user running it inside a pipeline must pass `-i` instead.

### Short forms that collide

Nothing collides on the agnostic surface, because it declares two options and they live on different commands. `-t`
means `--transport` on `sle mcp` alone, and `-i` means `--input-string` on `sle get checksum` alone. A system group may
bind either letter to something else, so quote the long form whenever a command from such a group is handed to a user.

---

## Command behavior and failure modes

### How a failure reaches the user

No command body in `interfaces/get.py` is wrapped in a catching decorator, and none carries an `except` handler, so
every failure the body raises leaves a Python traceback and a non-zero exit. This is the sharpest contrast with the tool
side, where every agnostic tool returns a string and all but two report failure with a leading `Error: ` prefix.
`/acquisition-system-setup` owns that tool-side convention. Ask the user to paste the traceback rather than the status.

| Path                                                   | Mechanism                         | Observable outcome                        |
|--------------------------------------------------------|-----------------------------------|-------------------------------------------|
| A group invoked with no subcommand                     | Click's `NoArgsIsHelpError`       | Group listing on stderr, exit 2           |
| A rejected `-t` value on `sle mcp`                     | Click's own parameter validation  | Usage message, exit 2                     |
| An unknown option on any node, `-h` included           | Click's own parser                | Usage message, exit 2                     |
| A non-ASCII string reaching `sle get checksum`         | `bytes(string, "ASCII")` raises   | `UnicodeEncodeError` traceback            |
| Any other error inside a `get` command body            | The exception propagates uncaught | Traceback, non-zero exit                  |
| `sle get controllers` finding no evaluable serial port | Warning logged, the body returns  | `No valid serial ports detected.`, exit 0 |
| `sle get ports` finding no port with a product ID      | Filtered out before printing      | Empty output, exit 0                      |
| `sle get unity` finding the bridge closed              | Warning logged, the body returns  | Unreachable warning, exit 0               |

There is no version option, no verbosity option, and no dry-run flag anywhere on the agnostic surface.

### `sle get zaber`

Calls `cross_system/zaber_bindings.py::discover_zaber_devices`, which scans every available serial port and prints the
heading `Device and Axis Information:` followed by the tabulated result. Both lines are echoed with `raw=True`, so the
console applies no log prefix and no line wrapping and the table arrives as the formatter produced it. Connection
errors met during the scan are logged at DEBUG level and never interrupt discovery, and a port that refuses connection
is listed as holding `No Devices`. Reading the table's device and axis semantics is owned by `/zaber-interface`.

### `sle get cameras`

Calls `ataraxis_video_system.discover_camera_ids`, the same function behind the `axvs` CLI, then splits the result by
`CameraInterfaces.OPENCV` and `CameraInterfaces.HARVESTERS` and prints each group under its own heading. The per-camera
line format is the one `video:cli-reference` documents for `axvs check devices`, and the leading number restarts at 1
inside each group, so it is never a camera index. Read `index=` instead. An OpenCV group is preceded by a warning that
the interface resolves no model and no serial number, which recommends `axvs run` for mapping indices to hardware.

`discover_camera_ids` returns the OpenCV cameras alone where the GenICam runtime is absent, which is every Intel Mac and
every macOS host running Python 3.14. Both `axvs check devices` and `sle get cameras` name that case with their own
`Harvesters camera discovery skipped.` line, so the two commands agree here. `sle get cameras` distinguishes the third
and fourth causes too, a present runtime with no configured CTI file and a configured path that no longer loads, both of
which `check_cti_file()` reports as `None`. That branch prints `Harvesters camera discovery skipped. No GenTL Producer
interface (.cti) file is configured.` and names `axvs cti set` and `AXVS_CTI_PATH` as the remedies. The `No
Harvesters-compatible cameras discovered.` warning therefore reaches the operator only for a genuinely empty GenTL bus,
so read it as that answer rather than as an ambiguous one.

### `sle get controllers`

Echoes `Evaluating serial ports at baudrate 115200, this may take a moment...` at INFO before it scans, because probing
every port takes seconds. The value comes from the module constant `_MICROCONTROLLER_BAUDRATE`, which is `115200` and
is not exposed as an option, so a controller answering on a different UART baudrate cannot be found through this
command. `discover_microcontrollers` drops every port that exposes no USB product identifier, probes the rest in
parallel across a process pool, and returns them in host enumeration order. Each surviving port prints as
`<n>: <port> -> <description> [<status>]`, numbered from 1.

| Status text                    | Meaning                                                           |
|--------------------------------|-------------------------------------------------------------------|
| `Microcontroller ID: <id>`     | The port answered the identification query with a controller ID   |
| `No microcontroller`           | The port was reachable and reported no microcontroller identifier |
| `Connection Failed: <message>` | The probe could not complete, and the message carries the reason  |

### `sle get ports`

Calls `ataraxis_transport_layer_pc.print_available_ports`, which enumerates every serial port, drops the ones exposing
no USB product identifier, and prints the survivors as `<n>: <device> -> <description>` numbered from 1. It opens no
port and identifies nothing connected to one, which makes it the fast complement to `sle get controllers`, and it is
the right command when a user needs a port name rather than a port's contents. It wraps its work in
`console.temporarily_enabled()`, so it prints even from a context that disabled the console and restores the previous
state afterward. It prints nothing at all when no port survives the filter, so empty output is the negative answer
rather than a hang.

### `sle get unity`

Builds a `UnityBridgeClient` against `http://127.0.0.1:8090/` with a five-second per-request timeout, probes it with
`is_reachable()`, and closes the client in a `finally` block. A reachable bridge prints `describe_status()` at SUCCESS,
in the form `Unity bridge: reachable | scene=<scene> | state=<state>`. An unreachable bridge prints a WARNING naming
the remedy, which is opening the Unity project in the editor, because the bridge starts with the editor.

The command probes twice, once for reachability and once for the status line, so a bridge that dies between the two
prints `Unity bridge: unreachable at <url>` at SUCCESS level. Read the message text rather than the log level.

### `sle get checksum`

Builds a `CRCCalculator` and prints `The CRC32-XFER checksum for the input string '<string>' is: <checksum>.` as one
line. `string_checksum` encodes through `bytes(string, "ASCII")`, so any non-ASCII character raises `UnicodeEncodeError`
uncaught. The command touches no hardware and no filesystem, which makes it the only agnostic command safe to run
while an acquisition session holds the host's ports.

---

## How the CLI diverges from the MCP path

The agnostic tool inventory, the `Error: ` return convention, and the two tools that depart from it are owned by
`/acquisition-system-setup`.

### CLI commands with no MCP equivalent

- `sle mcp` starts the server, so it is CLI-only by definition.
- `sle get cameras` has no tool on this server. Camera discovery reaches an agent through the `axvs` server instead,
  which `video:camera-setup` owns.
- `sle get controllers` has no tool on this server. Microcontroller discovery reaches an agent through the `axci`
  server instead, which `communication:microcontroller-setup` owns.
- `sle get ports` has no tool on this server. The closest agent-reachable substitute is the `axci` microcontroller
  discovery tool, which evaluates the same filtered port set and reports strictly more per port.

### MCP tools with no CLI equivalent

- **Every per-device Zaber surface.** `get_zaber_device_settings_tool`, `set_zaber_device_setting_tool`, and
  `validate_zaber_configuration_tool` have no CLI counterpart. `sle get zaber` enumerates devices and reads nothing out
  of any device's non-volatile memory, so reading, writing, and validating a device's configuration is tool-only.
  `/zaber-interface` owns all three.
- **The storage probe.** `check_mount_accessibility_tool` has no CLI counterpart. No `sle get` command touches the
  filesystem. `/acquisition-system-setup` owns the mount prerequisites.

### Where a paired command and tool differ

| CLI command        | Nearest MCP tool          | Divergence                                                                                                                                                                           |
|--------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `sle get zaber`    | `get_zaber_devices_tool`  | Same scan and same formatter. The command prints a heading and the table raw, where the tool returns the table alone as a string and catches every exception into an `Error:` prefix |
| `sle get unity`    | `check_unity_bridge_tool` | The command probes twice and downgrades an unreachable bridge to a warning naming the remedy. The tool probes once, returns the status line either way, and names no remedy          |
| `sle get checksum` | `get_checksum_tool`       | Same calculator and same value. The command prompts when `-i` is omitted, where the tool takes the string as a required argument, and the two wrap the number in different sentences |

### The three rules behind the table

1. **Ownership split.** Camera and microcontroller discovery live on the `axvs` and `axci` servers, so this server
   exposes no tool for either while the CLI carries a command for both. Reading a command name as a tool name fails on
   half of the `get` group.
2. **Depth.** The CLI discovers and the tools also read, write, and validate per-device state. Every configuration
   surface and the filesystem probe are tool-only, and every one of them concerns Zaber devices or storage mounts.
3. **Reach.** No agnostic command and no agnostic tool takes a host. Both address the machine that hosts the server.

---

## Fallback: what to tell a user when MCP is unavailable

Confirm the server is genuinely unrecoverable through `/experiment-mcp-environment-setup` before handing one of these
over.

| Blocked MCP tool          | Tell the user to run           |
|---------------------------|--------------------------------|
| `get_zaber_devices_tool`  | `sle get zaber`                |
| `check_unity_bridge_tool` | `sle get unity`                |
| `get_checksum_tool`       | `sle get checksum -i <string>` |

Three caveats. Every agnostic command prints to a terminal and returns nothing machine-readable, so ask for the output
pasted verbatim rather than summarized. The first two substitutes above touch the host's serial ports and the loopback
bridge, so hand those over only when no acquisition session is running. `sle get checksum` touches neither, and is safe
to run while a session holds the host's ports. When `sle mcp` is down but the `axvs` and `axci` servers are up, prefer
their discovery tools over `sle get cameras` and `sle get controllers`, because those two commands were never the agent
path for that hardware.

Everything else genuinely blocks until the server is back. That covers reading a Zaber device's non-volatile memory,
writing a setting into it, validating a device's configuration against the binding library, and probing a storage path
for existence and write access. Say so plainly rather than improvising a substitute out of a `get` command that does
something adjacent.

---

## Related skills

The `video:` and `communication:` entries below resolve through the ataraxis marketplace. Every other entry resolves
inside the sollertia marketplace.

| Skill                                  | Relationship                                                                   |
|----------------------------------------|--------------------------------------------------------------------------------|
| `mesoscope:mesoscope-vr-cli-reference` | Owns every `sle mesoscope` command and option this skill deliberately excludes |
| `/experiment-mcp-environment-setup`    | Owns the `sle --help` smoke test, the `sle mcp` transports, and MCP recovery   |
| `/acquisition-system-setup`            | Owns the discovery workflow, the tool inventory, and the `Error:` convention   |
| `/zaber-interface`                     | Owns the three Zaber tools with no CLI surface and the per-device semantics    |
| `/vr-driver-interface`                 | Owns the driver side of the Unity bridge `sle get unity` probes                |
| `/library-extension`                   | Owns the `_register_subcommands()` seam that registers a new system's group    |
| `/system-health-check`                 | Context: the pre-session sweep these discovery commands feed                   |
| `/pipeline`                            | Context: where hardware discovery sits in the end-to-end pipeline              |
| `unity:unity-mcp-environment-setup`    | Owns the editor-side relay `sle get unity` reaches over loopback               |
| `video:cli-reference`                  | Owns the camera line format `sle get cameras` reproduces from `axvs`           |
| `communication:microcontroller-setup`  | Owns the `axci` tools paired with `sle get controllers` and `sle get ports`    |

---

## Verification checklist

```text
Tool-settled (run `rg -n '.{121,}' <file>` and `wc -l <file>`):
- [ ] All lines at or under 120 characters (tables and code blocks may exceed for clarity)
- [ ] SKILL.md under 500 lines
- [ ] Every code fence carries a language identifier

Answering a CLI question, reader-judged:
- [ ] Answered from this skill or from sle COMMAND --help, never from memory
- [ ] Quoted the long option form, and never presented -h as a help alias
- [ ] Named the MCP tool to which the command maps, or said plainly that none exists on this server
- [ ] Routed any sle mesoscope question to mesoscope:mesoscope-vr-cli-reference instead of answering it
- [ ] Every sle mesoscope mention is the extension point or a handoff reference (rg -n 'sle mesoscope ' <file>)
- [ ] Invoked no sle command other than --help

Handing a user a CLI command, reader-judged:
- [ ] Confirmed the MCP server is genuinely unrecoverable through experiment-mcp-environment-setup first
- [ ] Printed the command for the user instead of running it
- [ ] Confirmed no acquisition session is running before recommending a command that opens serial ports
- [ ] Asked for the terminal output pasted verbatim, since no agnostic command prints machine-readable output
- [ ] Named axvs check devices instead when the absent GenICam runtime had to be distinguished
```
