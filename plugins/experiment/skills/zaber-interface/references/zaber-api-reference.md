# Zaber motor API reference

Complete API reference for the Zaber motor binding classes used in sollertia-experiment.

---

## Core imports

```python
from sollertia_experiment.cross_system.zaber_bindings import (
    ZaberConnection,
    ZaberDevice,
    ZaberAxis,
    CRCCalculator,
    discover_zaber_devices,
    get_zaber_devices_info,
)
```

The `__all__` list of `cross_system/__init__.py` re-exports `ZaberAxis`, `ZaberConnection`, `CRCCalculator`,
`discover_zaber_devices`, `get_zaber_devices_info`, `get_zaber_device_settings`, `set_zaber_device_setting`, and
`validate_zaber_device_configuration` from the `sollertia_experiment.cross_system` package. `ZaberDevice`,
`ZaberDeviceSettings`, and `ZaberValidationResult` are importable only from the `zaber_bindings` submodule, and a
`ZaberDevice` is normally reached through `ZaberConnection.get_device()` (`cross_system/zaber_bindings.py`).

---

## ZaberConnection class

Manages a serial USB port and all Zaber devices available through that port.

### Constructor

```python
class ZaberConnection:
    def __init__(self, port: str) -> None
```

**Parameters:**

| Parameter | Type  | Required | Description                             |
|-----------|-------|----------|-----------------------------------------|
| `port`    | `str` | Yes      | Serial port path (e.g., `/dev/ttyUSB0`) |

**Raises:** `TypeError` if port is not a string.

> Serial-port paths are OS-specific, appearing as `/dev/ttyUSB0` on Linux, `COMx` on Windows, and
> `/dev/tty.usbserial-*` on macOS. The examples in this reference use the Linux form, and the host reports the
> path to use.

**Notes:**
- The constructor stores the port name only. Call `connect()` to open the port (`cross_system/zaber_bindings.py`)
- `ZaberConnection` opens the port with `direct=False`, so Zaber Launcher can share it between applications.
  Without Zaber Launcher running, a second instance opening the same port fails

### Methods

All three methods are defined on `ZaberConnection` in `cross_system/zaber_bindings.py`.

| Method         | Returns       | Description                                     |
|----------------|---------------|-------------------------------------------------|
| `connect()`    | `None`        | Opens the port and wraps every detected device  |
| `disconnect()` | `None`        | Shuts the devices down and closes the port      |
| `get_device()` | `ZaberDevice` | Returns a device interface by daisy-chain index |

`connect()` returns immediately when the port is already open, and `disconnect()` returns immediately when it is not.
`connect()` rebuilds the internal device tuple after each successful device construction, so a failure partway through
the daisy chain still leaves the already-constructed devices reachable for release. On failure it calls
`_release_runtime_assets(shutdown_devices=False)` and re-raises. That path deliberately skips parking, because parking
commits the current position and blocks every motion command until the motor is unparked, and the operator may need to
move the stage by hand to free a head-fixed animal (`cross_system/zaber_bindings.py`).

`_release_runtime_assets` isolates each device shutdown through `run_shutdown_step` and closes the port from a
`finally` block, so one unresponsive controller still leaves the other devices and the port released
(`cross_system/zaber_bindings.py`). `ZaberConnection.__del__` runs the same release when the instance is still
connected at garbage-collection time.

### Properties

| Property       | Type   | Description                                                                |
|----------------|--------|----------------------------------------------------------------------------|
| `is_connected` | `bool` | Probes the port with `detect_devices()` and downgrades the flag on failure |

### get_device method

```python
def get_device(self, index: int) -> ZaberDevice
```

**Parameters:**

| Parameter | Type  | Description                                               |
|-----------|-------|-----------------------------------------------------------|
| `index`   | `int` | Zero-based index in daisy-chain (0 = closest to USB port) |

**Returns:** `ZaberDevice` instance for the specified controller.

**Raises:** `ConnectionError` if not connected to the port.

### Lifecycle

```text
__init__(port)  ──►  Port name stored, no port open and no devices available
connect()       ──►  Port open, every detected device wrapped in a ZaberDevice
get_device(i)   ──►  The ZaberDevice at daisy-chain index i
disconnect()    ──►  Every device shut down, port closed, resources released
```

---

## ZaberDevice class

Manages a Zaber controller that controls a single motor axis.

### Constructor

```python
class ZaberDevice:
    def __init__(self, device: Device) -> None
```

**Parameters:**

| Parameter | Type     | Description                                  |
|-----------|----------|----------------------------------------------|
| `device`  | `Device` | zaber-motion Device instance from connection |

**Notes:**
- Users do not instantiate this class directly. Use `ZaberConnection.get_device()`
- Only single-axis controllers are supported

**Raises:**
- `ValueError` if the device manages a number of axes other than 1
- `ValueError` if `USER_DATA_0` disagrees with the CRC32-XFER checksum of the device label
- `ValueError` if the operator declines the confirmation prompt raised for an unsafe, improperly shut down device

### Configuration validation

`ZaberDevice.__init__` in `cross_system/zaber_bindings.py` runs these steps in order:

1. **Axis count**: must be exactly 1, checked before the `ZaberAxis` is built
2. **Axis construction**: wraps axis 1, which validates the three predefined positions against the motion limits
3. **Checksum**: `USER_DATA_0` must equal the CRC32-XFER checksum of the device label
4. **Shutdown state**: when `shutdown_flag == 0` and `unsafe_flag == 1`, the constructor echoes a WARNING and blocks
   on `request_required_confirmation("Proceed with initializing this motor?")`
5. **Flag reset**: writes `USER_DATA_1 = 0`, so a runtime that aborts before shutdown stays detectable

### Methods

| Method       | Returns | Description                                      |
|--------------|---------|--------------------------------------------------|
| `shutdown()` | `None`  | Shuts the axis down and writes the shutdown flag |

`shutdown()` writes `USER_DATA_1 = 1` only when the motor is parked within `_PARK_POSITION_TOLERANCE` of the park
position stored in the same memory, and writes `0` with a WARNING for a motor resting anywhere else.

### Properties

| Property | Type        | Description                    |
|----------|-------------|--------------------------------|
| `axis`   | `ZaberAxis` | Interface for the motor (axis) |

---

## ZaberAxis class

Interfaces with a Zaber motor for motion control.

### Constructor

```python
class ZaberAxis:
    def __init__(self, motor: Axis) -> None
```

**Parameters:**

| Parameter | Type   | Description                            |
|-----------|--------|----------------------------------------|
| `motor`   | `Axis` | zaber-motion Axis instance from device |

**Notes:**
- Users do not instantiate this class directly. Access it through `ZaberDevice.axis`
- The constructor reads park, maintenance, and mount from `USER_DATA_11`, `USER_DATA_12`, and `USER_DATA_13`, and
  reads the two motion limits from the axis (`ZaberAxis.__init__` in `cross_system/zaber_bindings.py`)

**Raises:** `ValueError` if any of the three predefined positions falls outside the motion limits.

### Methods

All seven methods are defined on `ZaberAxis` in `cross_system/zaber_bindings.py`.

| Method           | Returns | Description                                                     |
|------------------|---------|-----------------------------------------------------------------|
| `home()`         | `None`  | Moves the motor to the home sensor position, non-blocking       |
| `move()`         | `None`  | Moves the motor to an absolute position, non-blocking           |
| `stop()`         | `None`  | Decelerates and stops the motor, emergency-safe                 |
| `park()`         | `None`  | Parks the motor and commits its position to non-volatile memory |
| `unpark()`       | `None`  | Unparks a parked motor so it accepts motion commands            |
| `shutdown()`     | `None`  | Stops a moving motor, then parks it                             |
| `get_position()` | `float` | Returns the current absolute position in native units           |

`park()` returns without acting while the motor is busy, and `unpark()` acts only on a parked motor. `shutdown()` is
idempotent through a private flag, returns early for an already parked motor, and stops a moving motor with a blocking
`stop` before parking it.

### Properties

| Property               | Type   | Description                                      |
|------------------------|--------|--------------------------------------------------|
| `is_homed`             | `bool` | True if motor has established home reference     |
| `is_parked`            | `bool` | True if motor is parked (cannot accept commands) |
| `is_busy`              | `bool` | True if motor is currently executing a command   |
| `park_position`        | `int`  | Predefined park position (native units)          |
| `mount_position`       | `int`  | Predefined mount position (native units)         |
| `maintenance_position` | `int`  | Predefined maintenance position (native units)   |

### home method

```python
def home(self) -> None
```

Moves the motor towards its home sensor until the sensor triggers. Establishes the reference point for all position
commands.

**Behavior:**
- Non-blocking. The call returns immediately and the motor moves asynchronously, so several motors can home in parallel
- Does nothing if the motor is parked or busy, and never auto-unparks a parked motor
- Poll `is_busy` to check completion

### move method

```python
def move(self, position: int) -> None
```

Moves the motor to the specified absolute position.

**Parameters:**

| Parameter  | Type  | Description                           |
|------------|-------|---------------------------------------|
| `position` | `int` | Target position in native motor units |

**Behavior:**
- Non-blocking. The call returns immediately and the motor moves asynchronously
- Does nothing if the motor is busy, un-homed, or parked
- Silently does nothing if the position falls outside `[minimum_limit, maximum_limit]`

### stop method

```python
def stop(self) -> None
```

Stops motor movement with deceleration.

**Behavior:**
- The only call that bypasses the communication pacing guard, for emergency use. It kicks the guard afterwards
- First call: decelerates and stops
- Second rapid call: immediate stop, with no deceleration
- Non-blocking

### Communication timing

`_COMMUNICATION_DELAY_MS` is `5`, the minimum delay in milliseconds separating consecutive interactions with the motor
hardware (`cross_system/zaber_bindings.py`). Every call other than `stop()` routes through the private generic
`_padded_method_call[T]`, which spins on a `Timeout` guard before the call and kicks the guard after it.

---

## Non-volatile memory settings

The private frozen dataclass `_ZaberSettings` maps every field name below to its Zaber `SettingConstants` member
(`cross_system/zaber_bindings.py`). Zaber controllers store the configuration in non-volatile USER_DATA
variables:

| Field                       | Variable     | Purpose                                       |
|-----------------------------|--------------|-----------------------------------------------|
| `checksum`                  | USER_DATA_0  | CRC32-XFER checksum of device label           |
| `shutdown_flag`             | USER_DATA_1  | 1 = proper shutdown, 0 = abnormal termination |
| `unsafe_flag`               | USER_DATA_10 | 1 = requires safe position for homing         |
| `axis_park_position`        | USER_DATA_11 | Park position in native units                 |
| `axis_maintenance_position` | USER_DATA_12 | Maintenance position in native units          |
| `axis_mount_position`       | USER_DATA_13 | Mount position in native units                |

`_ZaberSettings` also maps `maximum_limit`, `minimum_limit`, and `position` to the non-USER_DATA constants
`LIMIT_MAX`, `LIMIT_MIN`, and `POS`. The shorter `park_position`, `maintenance_position`, and `mount_position` names
belong to the
`ZaberDeviceSettings` snapshot and to the `setting` argument of `set_zaber_device_setting`, not to
`_ZaberSettings`, whose fields carry the `axis_` prefix.

**Understanding shutdown_flag vs unsafe_flag:**

- **shutdown_flag**: Managed during runtime. Set to `0` at startup, set to `1` during proper shutdown. If a motor with
  `unsafe_flag=1` has `shutdown_flag=0`, the system will prompt for manual verification before homing. To recover from
  improper shutdown, have the user verify the motor is safe, then set `shutdown_flag` to `1`.

- **unsafe_flag**: Set once during initial hardware setup. Reflects whether the motor's physical mounting allows it to
  be positioned in a way that makes homing dangerous (e.g., could cause collision). This flag should NOT be modified
  to work around improper shutdown. Use `shutdown_flag` instead.

`_PARK_POSITION_TOLERANCE` is `100.0` native units, the largest deviation from the stored park position that a
shutting-down device still records as properly parked (`cross_system/zaber_bindings.py`).

### Motion limit settings

| Setting         | Zaber constant | Purpose                                    |
|-----------------|----------------|--------------------------------------------|
| `maximum_limit` | LIMIT_MAX      | Maximum allowed position relative to home  |
| `minimum_limit` | LIMIT_MIN      | Minimum allowed position relative to home  |
| `position`      | POS            | Current absolute position relative to home |

---

## CRCCalculator class

Calculates CRC32-XFER checksums for device label validation.

### Constructor

```python
class CRCCalculator:
    def __init__(self) -> None
```

`CRCCalculator.__init__` builds a `crc.Configuration` with `width=32`, `polynomial=0x000000AF`, `init_value=0`,
`final_xor_value=0`, `reverse_input=False`, and `reverse_output=False` (`cross_system/zaber_bindings.py`).
`string_checksum` encodes the string as ASCII before checksumming it. The module-level `_crc_calculator` instance
backs the checksum verification `ZaberDevice` performs.

### Methods

| Method            | Returns | Description                               |
|-------------------|---------|-------------------------------------------|
| `string_checksum` | `int`   | Calculates CRC32-XFER checksum for string |

### string_checksum method

```python
def string_checksum(self, string: str) -> int
```

**Parameters:**

| Parameter | Type  | Description                    |
|-----------|-------|--------------------------------|
| `string`  | `str` | Input string (typically label) |

**Returns:** Integer CRC32-XFER checksum value.

**Example:**

```python
from sollertia_experiment.cross_system import CRCCalculator

calculator = CRCCalculator()
checksum = calculator.string_checksum("StageA")
print(f"Checksum for 'StageA': {checksum}")
# Use this value to configure USER_DATA_0 on the motor controller
```

`StageA` is a placeholder device label. For the labels the current worked example uses, see `mesoscope:mesoscope-vr`.

---

## Discovery functions

### discover_zaber_devices

Scans all serial ports and prints discovered device information.

```python
def discover_zaber_devices() -> None
```

**Notes:** Prints the formatted table through `console.echo(raw=True)`. A port that raises during the probe is logged
at DEBUG and listed as having "No Devices" (`cross_system/zaber_bindings.py`). Use `get_zaber_devices_info()`
for programmatic access. The `sle get zaber` CLI command wraps this function and takes no options
(the `get_zaber_devices` command in `interfaces/get.py`).

### get_zaber_devices_info

Scans all serial ports and returns formatted device information.

```python
def get_zaber_devices_info() -> str
```

**Returns:** Formatted table string containing port, device, and axis information.

**Notes:** Runs the same scan as `discover_zaber_devices` and backs the MCP tool `get_zaber_devices_tool()`
(`cross_system/zaber_bindings.py`).

---

## Configuration functions

### get_zaber_device_settings

Reads all configuration settings from a device's non-volatile memory.

```python
def get_zaber_device_settings(port: str, device_index: int) -> ZaberDeviceSettings
```

**Parameters:**

| Parameter      | Type  | Description                                          |
|----------------|-------|------------------------------------------------------|
| `port`         | `str` | Serial port path (e.g., `/dev/ttyUSB0`)              |
| `device_index` | `int` | Zero-based index in daisy-chain (0 = closest to USB) |

**Returns:** `ZaberDeviceSettings` dataclass containing:

| Attribute              | Type    | Source       |
|------------------------|---------|--------------|
| `device_label`         | `str`   | device.label |
| `axis_label`           | `str`   | axis.label   |
| `checksum`             | `int`   | USER_DATA_0  |
| `shutdown_flag`        | `int`   | USER_DATA_1  |
| `unsafe_flag`          | `int`   | USER_DATA_10 |
| `park_position`        | `int`   | USER_DATA_11 |
| `maintenance_position` | `int`   | USER_DATA_12 |
| `mount_position`       | `int`   | USER_DATA_13 |
| `limit_min`            | `float` | LIMIT_MIN    |
| `limit_max`            | `float` | LIMIT_MAX    |
| `current_position`     | `float` | POS          |

**Raises:**

- `ConnectionError`: If unable to connect to the specified port.
- `IndexError`: If device_index is out of range for the connected devices.

### set_zaber_device_setting

Writes a single setting to device non-volatile memory with validation.

```python
def set_zaber_device_setting(
    port: str,
    device_index: int,
    setting: str,
    value: int | str
) -> str
```

**Parameters:**

| Parameter      | Type         | Description                                              |
|----------------|--------------|----------------------------------------------------------|
| `port`         | `str`        | Serial port path                                         |
| `device_index` | `int`        | Zero-based index in daisy-chain                          |
| `setting`      | `str`        | Setting name (see table below)                           |
| `value`        | `int \| str` | Value to write (int for positions/flags, str for labels) |

**Valid settings:**

| Setting                | Type  | Validation                            |
|------------------------|-------|---------------------------------------|
| `park_position`        | `int` | Must be within [limit_min, limit_max] |
| `maintenance_position` | `int` | Must be within [limit_min, limit_max] |
| `mount_position`       | `int` | Must be within [limit_min, limit_max] |
| `shutdown_flag`        | `int` | Must be 0 or 1                        |
| `unsafe_flag`          | `int` | Must be 0 or 1 (rarely modified)      |
| `device_label`         | `str` | Auto-updates checksum                 |
| `axis_label`           | `str` | Optional, no validation               |

**Returns:** A success message of the form `"<setting>: <old> -> <new>"`. A `device_label` write appends the new
checksum to that message (the label branches of `set_zaber_device_setting` in `cross_system/zaber_bindings.py`).

**Raises:**

- `ConnectionError`: If unable to connect to the specified port.
- `IndexError`: If device_index is out of range.
- `TypeError`: If the value type does not match the setting (a non-string label, or a noninteger position or flag).
- `ValueError`: If the setting name is invalid or the value is out of range.

**Notes:** A `device_label` write also rewrites `USER_DATA_0` with the checksum of the new label. When the checksum
write fails, the function best-effort restores the previous label and raises `ValueError` naming both outcomes, so the
label and the checksum are brought back into agreement by re-running the same write. An `axis_label` write never
touches the checksum. Passing `checksum` raises `ValueError`, because the binding library owns that value
(`cross_system/zaber_bindings.py`).

**Note on axis_label:**
- The `axis_label` is optional and typically unused for Zaber motors. A missing axis_label is not a configuration issue.
- Axis labels are primarily used for third-party motors where the label reflects the motor name.
- For Zaber single-axis controllers, the `device_label` drives checksum validation and can repeat across a motor
  group, because the binding library selects each device by its daisy-chain index.

### validate_zaber_device_configuration

Validates device configuration for use with the binding library.

```python
def validate_zaber_device_configuration(port: str, device_index: int) -> ZaberValidationResult
```

**Parameters:**

| Parameter      | Type  | Description                     |
|----------------|-------|---------------------------------|
| `port`         | `str` | Serial port path                |
| `device_index` | `int` | Zero-based index in daisy-chain |

**Returns:** `ZaberValidationResult` dataclass containing:

| Attribute         | Type              | Description                                          |
|-------------------|-------------------|------------------------------------------------------|
| `is_valid`        | `bool`            | Overall validation result                            |
| `checksum_valid`  | `bool`            | Whether stored checksum matches calculated           |
| `positions_valid` | `bool`            | Whether all positions are within motion limits       |
| `errors`          | `tuple[str, ...]` | Critical issues preventing use with binding library  |
| `warnings`        | `tuple[str, ...]` | Non-critical issues that may affect device operation |

**Checks, in order** (`validate_zaber_device_configuration` in `cross_system/zaber_bindings.py`):

1. An unset device label is rejected outright and marks the checksum invalid, because the checksum of an empty label
   equals the factory `USER_DATA_0` value of 0 and would otherwise pass
2. The stored checksum is compared against the CRC32-XFER checksum of the label
3. Park, maintenance, and mount are each bounds-checked against `limit_min` and `limit_max`
4. A device with `shutdown_flag == 0` and `unsafe_flag == 1` adds a warning rather than an error

**Raises:**

- `ConnectionError`: If unable to connect to the specified port.
- `IndexError`: If device_index is out of range.

---

## Dependencies

### External requirements

| Dependency       | Required | Purpose                                                     |
|------------------|----------|-------------------------------------------------------------|
| zaber-motion     | Yes      | Python bindings for the Zaber ASCII protocol                |
| USB serial port  | Yes      | Physical connection to Zaber controllers                    |
| Port permissions | Yes      | OS-level serial-port access (e.g. `dialout` group on Linux) |

### Python requirements

Python package versions are declared and pinned in the `dependencies` array of `pyproject.toml`, which is the
authoritative source and is enforced at install time. The Zaber stack imports `zaber-motion`, `crc`, `tabulate`,
`ataraxis-time`, and `ataraxis-base-utilities`. Consult that file for the current pins.

---

## Code examples

### Basic motor control

```python
from sollertia_experiment.cross_system.zaber_bindings import ZaberConnection

# Connect to motor group
connection = ZaberConnection(port="/dev/ttyUSB0")
connection.connect()

# Get motor interface
motor = connection.get_device(index=0).axis

# Home motor (establishes reference)
motor.unpark()
motor.home()
while motor.is_busy:
    pass  # Wait for homing to complete

# Move to mount position
motor.move(position=motor.mount_position)
while motor.is_busy:
    pass

# Park and disconnect
motor.park()
connection.disconnect()
```

### Multi-motor coordination

```python
from sollertia_experiment.cross_system.zaber_bindings import ZaberConnection

# Connect to daisy-chained motors
connection = ZaberConnection(port="/dev/ttyUSB0")
connection.connect()

# Get all motor axes
z_axis = connection.get_device(index=0).axis
pitch_axis = connection.get_device(index=1).axis
roll_axis = connection.get_device(index=2).axis

# Unpark all motors
z_axis.unpark()
pitch_axis.unpark()
roll_axis.unpark()

# Home all motors in parallel (non-blocking)
z_axis.home()
pitch_axis.home()
roll_axis.home()

# Wait for all to complete
while z_axis.is_busy or pitch_axis.is_busy or roll_axis.is_busy:
    pass

# Move all motors to park position in parallel
z_axis.move(position=z_axis.park_position)
pitch_axis.move(position=pitch_axis.park_position)
roll_axis.move(position=roll_axis.park_position)

# Wait and park
while z_axis.is_busy or pitch_axis.is_busy or roll_axis.is_busy:
    pass

z_axis.park()
pitch_axis.park()
roll_axis.park()

connection.disconnect()
```

### Position snapshot and restoration

```python
from sollertia_experiment.cross_system.zaber_bindings import ZaberConnection

# Connect
connection = ZaberConnection(port="/dev/ttyUSB0")
connection.connect()
motor = connection.get_device(index=0).axis

# Take a position snapshot (the consuming system persists this however it tracks per-session state)
saved_position = int(motor.get_position())

# Later: restore from the snapshot
motor.unpark()
motor.move(position=saved_position)
while motor.is_busy:
    pass
motor.park()

connection.disconnect()
```

### Emergency stop

```python
from sollertia_experiment.cross_system.zaber_bindings import ZaberConnection

connection = ZaberConnection(port="/dev/ttyUSB0")
connection.connect()
motor = connection.get_device(index=0).axis

motor.unpark()
motor.move(position=50000)  # Start moving

# Emergency stop (can be called anytime, bypasses timing guards)
motor.stop()  # First call: decelerate and stop
motor.stop()  # Second rapid call: immediate stop

connection.disconnect()
```

---

## Binding class patterns

When implementing Zaber motor support in a binding class, follow these patterns:

### Basic structure

```python
class SystemZaberMotors:
    """Manages Zaber motor groups for the acquisition system.

    Args:
        zaber_configuration: Motor configuration from system config.
        zaber_positions: Previous session positions or None for defaults.

    Attributes:
        _connection: ZaberConnection for the motor group.
        _axis: ZaberAxis for the motor.
    """

    def __init__(
        self,
        zaber_configuration: SystemExternalAssets,
        zaber_positions: SystemZaberPositions | None,
    ) -> None:
        # Initialize connection
        self._connection: ZaberConnection = ZaberConnection(
            port=zaber_configuration.primary_motor_port
        )

        # Connect and get device/axis
        self._connection.connect()
        self._axis: ZaberAxis = self._connection.get_device(index=0).axis

        # Store previous positions for restoration
        self._previous_positions = zaber_positions

    def restore_position(self) -> None:
        """Restores motors to previous session positions."""
        self.unpark_motors()

        if self._previous_positions is not None:
            self._axis.move(position=self._previous_positions.motor_position)
        else:
            self._axis.move(position=self._axis.mount_position)

        self.wait_until_idle()
        self.park_motors()

    def wait_until_idle(self) -> None:
        """Blocks until all motors finish moving."""
        while self._axis.is_busy:
            pass

    def disconnect(self) -> None:
        """Shuts down motors and closes connection."""
        self._connection.disconnect()

    def park_motors(self) -> None:
        """Parks all motors to prevent accidental movement."""
        self._axis.park()

    def unpark_motors(self) -> None:
        """Unparks motors to allow movement commands."""
        self._axis.unpark()
```

### Key patterns

| Pattern               | Purpose                                         |
|-----------------------|-------------------------------------------------|
| Park/unpark guards    | Prevent accidental movement during idle periods |
| Position restoration  | Maintain consistent animal positioning          |
| Wait until idle       | Coordinate multi-motor movements                |
| Destructor disconnect | Ensure proper shutdown on garbage collection    |

---

## Configuration requirements

Motor configuration must be defined in the consuming acquisition system's configuration module before
implementation. Each acquisition system defines its own configuration class exposing the per-motor-group serial
ports; consult that system's own skill for the concrete class.

### Required configuration fields

| Field    | Type  | Description                             |
|----------|-------|-----------------------------------------|
| `*_port` | `str` | Serial port path (e.g., `/dev/ttyUSB0`) |

### Configuration dataclass pattern

```python
@dataclass()
class SystemExternalAssets:
    """External asset configuration for the acquisition system."""

    primary_motor_port: str = "/dev/ttyUSB0"
    """Serial port for the first motor group."""

    secondary_motor_port: str = "/dev/ttyUSB1"
    """Serial port for the second motor group."""
```

### Position data pattern

```python
@dataclass()
class SystemZaberPositions:
    """Stores motor positions for session restoration (one int field per managed motor axis)."""

    z: int = 0
    """Z-axis position in native motor units."""

    pitch: int = 0
    """Pitch-axis position in native motor units."""

    roll: int = 0
    """Roll-axis position in native motor units."""

    # ... one int field per additional motor axis
```
