---
name: zaber-interface
description: >-
  Guides implementation of Zaber motor interfaces using the zaber-motion library. Covers motor
  discovery, position management, safety patterns, and binding class patterns. Use when adding
  Zaber motor support to any acquisition system or troubleshooting motor connectivity.
user-invocable: false
---

# Zaber motor interface

Guides the implementation of Zaber motor interfaces using the zaber-motion library. This is the
platform-general Zaber hardware subsystem — the `ZaberConnection` / `ZaberDevice` / `ZaberAxis` stack in
`sollertia_experiment/cross_system/zaber_bindings.py` is system-agnostic and consumed by any
acquisition system's binding layer (currently Mesoscope-VR's `ZaberMotors`).

---

## Scope

**Covers:**
- Discovering Zaber motors and recording their port and daisy-chain assignments
- Reading, modifying, and validating motor configuration in non-volatile memory (positions, flags, labels, checksum)
- The `ZaberConnection` / `ZaberDevice` / `ZaberAxis` API hierarchy and its safety patterns
- Composing Zaber motors into an acquisition system's binding class

**Does not cover:**
- Mesoscope-VR-specific integration — modifying `MesoscopeVRAssets` or extending `ZaberMotors`
  (see `mesoscope:mesoscope-vr`)
- The platform-general pattern for composing a Zaber subsystem into a binding class (see `/acquisition-system-design`)
- Reading or writing the `ZaberPositions` session snapshot (see `mesoscope:mesoscope-vr-snapshots`)
- The `zaber-motion` library internals (third-party; consult its own documentation)

---

## When to use this skill

Use this skill when:

- Adding Zaber motor support to an acquisition system
- Troubleshooting Zaber motor connectivity issues
- Verifying motor configuration before runtime
- Understanding the ZaberConnection/ZaberDevice/ZaberAxis API hierarchy
- Configuring motor positions in non-volatile memory

For Mesoscope-VR-specific integration (modifying `MesoscopeVRAssets`, extending the `ZaberMotors`
binding class), use `mesoscope:mesoscope-vr`. For the platform-general pattern by which an acquisition system
composes Zaber motors into its binding layer, see `/acquisition-system-design`.

---

## Verification requirements

**Before writing any Zaber code, verify the hardware is connected and accessible.**

### Step 0: Hardware verification

Use the sollertia-experiment MCP server for Zaber discovery. Start the server with:
```bash
sle mcp
```

**MCP tool for verification:**

| Tool                     | Purpose                                      |
|--------------------------|----------------------------------------------|
| `get_zaber_devices_tool` | Discovers Zaber devices and their ports/axes |

**Verification workflow:**

1. **Discover Zaber devices**: Run `get_zaber_devices_tool()` to identify connected motors
2. **Note port assignments**: Record which serial port the discovery tool reports for each motor group
3. **Verify device order**: Confirm daisy-chain order matches expected configuration

**Expected output from `get_zaber_devices_tool()`:**
```text
+----------------+------------+-------+---------+-------------+---------+-------------+
|      Port      | Device Num |  ID   |  Label  |    Name     | Axis ID | Axis Label  |
+----------------+------------+-------+---------+-------------+---------+-------------+
| /dev/ttyUSB0   |     1      | 30341 | HeadBar |  X-LDA025A  |    1    |      Z      |
|                |     2      | 30341 |         |  X-LDA025A  |    1    |    Pitch    |
|                |     3      | 30341 |         |  X-LDA025A  |    1    |    Roll     |
+----------------+------------+-------+---------+-------------+---------+-------------+
```

> **OS note.** Serial-port paths are OS-specific. This skill shows the Linux form (`/dev/ttyUSB0`); on Windows
> the same port appears as a `COM3`-style name, on macOS as `/dev/tty.usbserial-XXXX`. Always use the path the
> discovery tool reports for the host.

If motors are not detected:
- Check USB connections and power supplies
- Verify the OS grants the current user access to the serial port. On Linux, add the user to the `dialout`
  group (`sudo usermod -a -G dialout $USER`, then re-login); on Windows or macOS, install the vendor's
  USB-serial driver and confirm no other application holds the port open
- Ensure motors are powered on before connecting USB
- Check for port conflicts with other applications

### Step 1: Content verification

| File                                                                            | What to Check                                                       |
|---------------------------------------------------------------------------------|---------------------------------------------------------------------|
| `sollertia-experiment/src/sollertia_experiment/cross_system/zaber_bindings.py`  | ZaberConnection/Device/Axis patterns                                |
| `sollertia-experiment/src/sollertia_experiment/mesoscope_vr/binding_classes.py` | Example/reference ZaberMotors binding class (Mesoscope-VR-specific) |
| `sollertia-experiment pyproject.toml`                                           | Current zaber-motion version dependency                             |

---

## Architecture overview

Zaber motor control uses a tri-class hierarchy:

```text
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        ZaberConnection (Port Level)                             │
│  ────────────────────────────────────────────────────────────────────────────── │
│  - Manages serial port connection                                               │
│  - Discovers all devices on the port                                            │
│  - Coordinates shutdown across all devices                                      │
│                                                                                 │
│     ┌─────────────────────────────────────────────────────────────────────────┐ │
│     │                    ZaberDevice (Controller Level)                       │ │
│     │  ─────────────────────────────────────────────────────────────────────  │ │
│     │  - Validates device configuration (checksum verification)               │ │
│     │  - Manages shutdown tracking in non-volatile memory                     │ │
│     │  - Exposes the axis (motor) interface                                   │ │
│     │                                                                         │ │
│     │     ┌─────────────────────────────────────────────────────────────────┐ │ │
│     │     │                   ZaberAxis (Motor Level)                       │ │ │
│     │     │  ────────────────────────────────────────────────────────────── │ │ │
│     │     │  - Executes motion commands (home, move, stop)                  │ │ │
│     │     │  - Manages predefined positions (park, mount, maintenance)      │ │ │
│     │     │  - Enforces communication timing and safety patterns            │ │ │
│     │     └─────────────────────────────────────────────────────────────────┘ │ │
│     └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Key relationships:**
- One `ZaberConnection` per serial port (USB cable)
- Multiple `ZaberDevice` instances per connection (daisy-chained motors)
- One `ZaberAxis` per device (single-axis controllers only)

---

## Motor discovery

Use the MCP tool `get_zaber_devices_tool()` to discover connected Zaber motors.

### Discovery output fields

| Field      | Description                                                          |
|------------|----------------------------------------------------------------------|
| Port       | Serial port path (e.g., `/dev/ttyUSB0`)                              |
| Device Num | Position in daisy-chain (1 = closest to USB)                         |
| ID         | Hardware device identifier code                                      |
| Label      | User-assigned label stored in non-volatile memory                    |
| Name       | Manufacturer model name                                              |
| Axis ID    | Axis number within the device (always 1 for single-axis controllers) |
| Axis Label | User-assigned axis label stored in non-volatile memory               |

### Daisy-chain ordering

Motors connected to the same serial port form a daisy-chain. The device number reflects physical position:

```text
USB Port ──► Device 1 ──► Device 2 ──► Device 3
             (index 0)    (index 1)    (index 2)
```

**Important:** When adding motors, document the expected daisy-chain order in configuration. Misordering causes motors
to move incorrectly.

---

## Position management

Zaber motors use predefined positions stored in non-volatile memory for safe operation.

### Position types

| Position    | Purpose                                      | When Used                      |
|-------------|----------------------------------------------|--------------------------------|
| Park        | Safe position for shutdown and storage       | System shutdown, storage       |
| Mount       | Position for mounting animal into enclosure  | Session start, animal mounting |
| Maintenance | Position for system maintenance and cleaning | Between sessions, maintenance  |

### Position storage

Positions are stored in non-volatile USER_DATA variables on each motor controller:

| Variable     | Purpose                                   |
|--------------|-------------------------------------------|
| USER_DATA_11 | Park position (native motor units)        |
| USER_DATA_12 | Maintenance position (native motor units) |
| USER_DATA_13 | Mount position (native motor units)       |

### Position restoration

A consuming acquisition system's binding class can restore motors to their previous-session positions from a
position snapshot the system provides and persists. This enables consistent positioning across sessions. For the
Mesoscope-VR implementation, see `mesoscope:mesoscope-vr` (the `ZaberMotors` consumer) and
`mesoscope:mesoscope-vr-snapshots` (the snapshot).

---

## Agentic configuration management

Use MCP tools to read and modify Zaber motor configuration stored in non-volatile memory.

### Available MCP tools

| Tool                                                                                  | Purpose                       |
|---------------------------------------------------------------------------------------|-------------------------------|
| `get_zaber_devices_tool()`                                                            | Discover connected motors     |
| `get_zaber_device_settings_tool(port, device_index)`                                  | Read device configuration     |
| `set_zaber_device_setting_tool(port, device_index, setting, value, *, confirm=False)` | Modify device setting         |
| `validate_zaber_configuration_tool(port, device_index)`                               | Validate device configuration |
| `get_checksum_tool(input_string)`                                                     | Calculate CRC32-XFER checksum |

### Configuration workflow

#### Reading current configuration

1. Discover devices: `get_zaber_devices_tool()`
2. Read settings: `get_zaber_device_settings_tool(port="/dev/ttyUSB0", device_index=0)`
3. Validate configuration: `validate_zaber_configuration_tool(port="/dev/ttyUSB0", device_index=0)`

#### Modifying configuration

**Safety protocol:**

1. Read current value using `get_zaber_device_settings_tool()`
2. Show user the current value and proposed change
3. Preview change: `set_zaber_device_setting_tool(..., confirm=False)`
4. Execute change: `set_zaber_device_setting_tool(..., confirm=True)`
5. Verify change: `get_zaber_device_settings_tool()`

### Configurable settings

| Setting                | Type  | Description                             | Constraints                  |
|------------------------|-------|-----------------------------------------|------------------------------|
| `park_position`        | `int` | Shutdown position (native units)        | Must be within motion limits |
| `maintenance_position` | `int` | Maintenance position (native units)     | Must be within motion limits |
| `mount_position`       | `int` | Animal mounting position (native units) | Must be within motion limits |
| `shutdown_flag`        | `int` | Proper shutdown indicator (see below)   | 0 or 1                       |
| `unsafe_flag`          | `int` | Requires safe position for homing       | 0 or 1 (rarely modified)     |
| `device_label`         | `str` | Device identifier                       | Auto-updates checksum        |
| `axis_label`           | `str` | Axis identifier (optional, see below)   | No constraints               |

### Read-only settings

| Setting            | Description                       |
|--------------------|-----------------------------------|
| `checksum`         | Auto-calculated from device_label |
| `limit_min`        | Hardware motion limit             |
| `limit_max`        | Hardware motion limit             |
| `current_position` | Live motor position               |

### Understanding shutdown_flag vs unsafe_flag

**shutdown_flag (USER_DATA_1):**
- Set to `1` during proper system shutdown, set to `0` at startup
- If a motor with `unsafe_flag=1` has `shutdown_flag=0`, the system prompts for manual verification before homing
- **This is the flag you typically manage** when recovering from improper shutdown (power loss, crash, etc.)
- To recover: manually verify the motor is in a safe position, then set `shutdown_flag` to `1`

**unsafe_flag (USER_DATA_10):**
- Indicates whether the motor can be positioned unsafely for homing (e.g., where homing could cause collision)
- **This flag is set during initial hardware setup** and reflects physical assembly constraints
- **Do NOT modify this flag** unless the physical hardware configuration has changed
- If a motor's physical mounting allows safe homing from any position, `unsafe_flag` should be `0`
- If a motor could be left in a position that makes homing dangerous, `unsafe_flag` should be `1`

**Recovery workflow for improper shutdown:**
1. User confirms the motor is physically positioned safely for homing
2. Agent sets `shutdown_flag` to `1` using `set_zaber_device_setting_tool`
3. System can now initialize normally without the improper shutdown warning

### Understanding axis_label vs device_label

**device_label:**
- Required for all motors. Used for checksum validation to verify the device is configured for the binding library.
- Examples: "HeadBar", "Wheel", "Lickport"

**axis_label:**
- Optional and typically unused for Zaber motors. A missing axis_label is not an issue.
- Axis labels are primarily used for third-party motors where the label reflects the specific motor name.
- For Zaber single-axis controllers, the device_label is sufficient for identification.
- Do not flag missing axis_label as a configuration problem.

### Initial device setup workflow

For new motors not yet configured for use with the binding library:

1. **Discover device**: `get_zaber_devices_tool()`
2. **Set device label**: `set_zaber_device_setting_tool(port, index, "device_label", "HeadBar", confirm=True)`
   (This automatically calculates and sets the checksum)
3. **Set axis label**: `set_zaber_device_setting_tool(port, index, "axis_label", "Z", confirm=True)`
4. **Set positions**: Configure park, maintenance, and mount positions
5. **Set unsafe flag** (if needed): Only set this during initial setup based on physical hardware constraints.
   Set to `1` if the motor can be positioned unsafely for homing (e.g., where homing could cause collision).
6. **Validate**: `validate_zaber_configuration_tool(port, index)`

### Improper shutdown recovery workflow

When a motor with `unsafe_flag=1` was not properly shut down:

1. **Read current settings**: `get_zaber_device_settings_tool(port, index)` to confirm `shutdown_flag=0`
2. **User verification**: Ask the user to physically verify the motor is in a safe position for homing
3. **Reset shutdown flag**: `set_zaber_device_setting_tool(port, index, "shutdown_flag", "1", confirm=True)`
4. **Validate**: `validate_zaber_configuration_tool(port, index)` should now show no warnings

**Important:** Never modify `unsafe_flag` to work around improper shutdown. The `unsafe_flag` reflects physical
hardware constraints and should only be changed if the hardware assembly changes.

---

## Safety patterns

### Park/unpark workflow

Motors use a parking mechanism to prevent accidental movement:

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                        Motor Safety State Machine                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌──────────┐                                                          │
│   │  PARKED  │ ◄──── park() ──── Motors cannot move                     │
│   └────┬─────┘                                                          │
│        │                                                                │
│    unpark()                                                             │
│        │                                                                │
│        ▼                                                                │
│   ┌──────────┐                                                          │
│   │ UNPARKED │ ◄──── Motors can execute commands                        │
│   └────┬─────┘                                                          │
│        │                                                                │
│    home() / move() / etc.                                               │
│        │                                                                │
│        ▼                                                                │
│   ┌──────────┐                                                          │
│   │  MOVING  │ ◄──── is_busy = True                                     │
│   └────┬─────┘                                                          │
│        │                                                                │
│    wait_until_idle()                                                    │
│        │                                                                │
│        ▼                                                                │
│   ┌──────────┐                                                          │
│   │   IDLE   │ ◄──── is_busy = False, ready for next command            │
│   └──────────┘                                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Critical pattern:** Always call `unpark_motors()` before movement and `park_motors()` after completion.

### Shutdown safety

Motors use non-volatile flags to detect improper shutdown:

| Flag          | Variable     | Purpose                                               |
|---------------|--------------|-------------------------------------------------------|
| Shutdown flag | USER_DATA_1  | Set to 1 on proper shutdown, 0 on startup             |
| Unsafe flag   | USER_DATA_10 | Indicates motor requires specific position for homing |

If a motor with `unsafe_flag=True` was not properly shut down, the system prompts for manual verification before
proceeding.

### Checksum validation

Each device stores a CRC32-XFER checksum of its label in USER_DATA_0. This validates that the motor is configured for
use with the binding library:

```python
# Calculate expected checksum
from sollertia_experiment.cross_system import CRCCalculator
calculator = CRCCalculator()
expected = calculator.string_checksum("HeadBar")  # Device label
```

Use the MCP tool `get_checksum_tool(input_string)` to calculate checksums for configuration.

---

## ZaberAxis API reference

See [references/zaber-api-reference.md](references/zaber-api-reference.md) for the complete API reference including:

- ZaberConnection constructor and methods
- ZaberDevice configuration validation
- ZaberAxis motion commands and properties
- Non-volatile memory settings structure
- Complete code examples

---

## Binding class patterns and configuration

When composing Zaber motors into an acquisition system's binding class, follow these established
binding-class patterns: park/unpark guards around every movement, position restoration from a
previous-session snapshot, a `wait_until_idle()` barrier before parking, and connection teardown
on disconnect. The consuming system supplies the motor port assignments and a position dataclass
(one field per managed motor axis).

For the full binding-class skeleton, the key-patterns table, and the configuration and position
dataclass patterns, see [references/zaber-api-reference.md](references/zaber-api-reference.md).

---

## Troubleshooting

### Motor not detected

1. Verify USB cable is connected and motor is powered
2. Confirm the host lists the serial port using an OS-appropriate method (Linux: `ls -la /dev/ttyUSB*`;
   Windows: Device Manager → Ports; macOS: `ls -la /dev/tty.usbserial-*`)
3. Ensure the user can access the port (Linux: add the user to the `dialout` group with
   `sudo usermod -a -G dialout $USER`, then re-login; Windows/macOS: install the vendor USB-serial driver)
4. Verify no other application is using the port
5. Run `get_zaber_devices_tool()` to check discovery

### Checksum validation failure

1. Motor not configured for use with binding library
2. Calculate expected checksum: `get_checksum_tool("DeviceLabel")`
3. Use Zaber Launcher to set USER_DATA_0 to calculated checksum
4. Verify device label matches expected value

### Improper shutdown warning

1. Motor was not shut down properly in previous session
2. Verify motor is positioned safely for homing
3. Confirm the "Proceed with initializing this motor?" prompt with `y` (the prompt has no default and
   re-prompts on any empty or unrecognized answer; entering `n`/`no` aborts initialization)
4. Or manually set USER_DATA_1 to 1 in Zaber Launcher

### Movement not executing

1. Verify motor is unparked: `is_parked` property
2. Verify motor is homed: `is_homed` property
3. Verify motor is idle: `is_busy` property
4. Check target position is within limits

### Daisy-chain order mismatch

1. Run `get_zaber_devices_tool()` to see actual order
2. Compare Device Num with expected configuration
3. Physically reorder cables if necessary
4. Update configuration to match actual order

---

## Related skills

| Skill                               | Relationship                                                                   |
|-------------------------------------|--------------------------------------------------------------------------------|
| `/acquisition-system-design`        | Platform-general pattern for composing a Zaber subsystem into a binding class  |
| `mesoscope:mesoscope-vr`            | Current consumer — composes `ZaberMotors` from `MesoscopeVRAssets`             |
| `mesoscope:mesoscope-vr-snapshots`  | Reads/writes the `ZaberPositions` snapshot this subsystem restores from        |
| `/acquisition-system-setup`         | Acquisition-system-level hardware discovery and verification                   |
| `/experiment-mcp-environment-setup` | Run first if the `sle mcp` server is not connected                             |
| `references/zaber-api-reference.md` | Complete `ZaberConnection` / `ZaberDevice` / `ZaberAxis` API and code examples |

---

## Verification checklist

Before integrating Zaber motors into an acquisition system:

```text
- [ ] Discovered motors using get_zaber_devices_tool()
- [ ] Recorded port assignments for each motor group
- [ ] Verified daisy-chain order matches expected configuration
- [ ] Calculated and verified checksum for each device label
- [ ] Confirmed motors have predefined positions in non-volatile memory
- [ ] Defined port assignment fields in the consuming system's configuration dataclass
- [ ] Implemented binding class with park/unpark safety patterns
- [ ] Added position snapshot and restoration support
- [ ] Integrated into the acquisition system's data-acquisition lifecycle (Mesoscope-VR ref: `mesoscope:mesoscope-vr`)
- [ ] MyPy strict passes
```
