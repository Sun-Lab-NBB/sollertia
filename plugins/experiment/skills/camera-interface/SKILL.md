---
name: sollertia-camera-interface
description: >-
  Documents how Sollertia experiment binding classes wrap ataraxis-video-system VideoSystem instances for
  the Mesoscope-VR acquisition system. Covers MesoscopeCameras configuration semantics, system ID
  allocation for face/body cameras, and binding class lifecycle. Use when adding or modifying camera
  acquisition for a Sollertia experiment. Delegates to ataraxis video skills for low-level VideoSystem API.
user-invocable: true
---

# Sollertia Camera Interface

Documents how Sollertia experiment binding classes wrap `ataraxis-video-system` for the Mesoscope-VR
acquisition system. Focuses exclusively on the Sollertia data class semantics and binding patterns.

For low-level VideoSystem API, camera discovery, encoding, and runtime testing, this skill delegates to
the `ataraxis@video` plugin skills.

---

## Scope

**Covers:**
- The `MesoscopeCameras` configuration dataclass and its fields
- System ID allocation for cameras in the Mesoscope-VR system
- Binding class lifecycle for cameras within a Sollertia acquisition system
- Where camera configuration lives in the Sollertia configuration tree

**Does not cover** (delegated to ataraxis):
- VideoSystem constructor parameters and lifecycle methods
- Camera discovery, runtime requirements, CTI configuration, encoder selection
- Interactive camera testing via MCP tools
- FFMPEG / GPU troubleshooting

---

## Required ataraxis skill handoffs

You MUST read the relevant ataraxis skill before writing or modifying any camera code. The Sollertia
binding layer is intentionally thin — all hardware interaction goes through ataraxis-video-system.

| Task                                       | Required ataraxis skill            |
|--------------------------------------------|------------------------------------|
| Discovering cameras / verifying hardware   | `ataraxis@video:camera-setup`      |
| Writing or modifying VideoSystem code      | `ataraxis@video:camera-interface`  |
| Diagnosing MCP server issues               | `ataraxis@video:mcp-environment-setup` |
| Understanding recorded log archive layout  | `ataraxis@video:log-input-format`  |

---

## Sollertia binding semantics

### MesoscopeCameras configuration dataclass

Defined in `sollertia_shared_assets.configuration.mesoscope_configuration.MesoscopeCameras`, this dataclass
captures the full per-camera configuration for the Mesoscope-VR system. It is loaded from disk as part of
`MesoscopeSystemConfiguration` and passed to the binding class at runtime.

| Field                       | Maps to ataraxis-video-system parameter |
|-----------------------------|------------------------------------------|
| `face_camera_index`         | `camera_index` for the face VideoSystem  |
| `body_camera_index`         | `camera_index` for the body VideoSystem  |
| `face_camera_quantization`  | `quantization_parameter` (face)          |
| `face_camera_preset`        | `encoder_speed_preset` (face)            |
| `body_camera_quantization`  | `quantization_parameter` (body)          |
| `body_camera_preset`        | `encoder_speed_preset` (body)            |

The `*_preset` fields are plain integers in YAML. They are coerced to `EncoderSpeedPresets` IntEnum members
inside the binding class — see `ataraxis@video:camera-interface` for the enum values.

The Mesoscope-VR system uses **Harvesters** as the camera backend (GenICam cameras). The binding class
hard-codes `camera_interface=CameraInterfaces.HARVESTERS`; this is not configurable per-system.

### System ID allocation

Each VideoSystem on the Mesoscope-VR system requires a unique `system_id` (uint8) for DataLogger routing.
The Sollertia mesoscope-vr binding layer reserves the following IDs:

| ID  | Camera        |
|-----|---------------|
| 51  | Face camera   |
| 62  | Body camera   |

When adding a new camera to the Mesoscope-VR system, allocate the next free ID in the 50–99 range. IDs
1–49 are reserved for non-camera hardware (microcontrollers, motors). See
`ataraxis@video:camera-interface` for the broader system ID allocation rules.

### Binding class lifecycle

The Sollertia binding wraps each VideoSystem in a small per-camera class with idempotent
`start()` / `start_saving()` / `stop()` methods. For the canonical pattern, see the
`MesoscopeVRSystem` source in `sollertia-experiment` and the binding class section of
`ataraxis@video:camera-interface`.

Key Sollertia-specific rules:

1. **Configuration injection.** The binding class accepts `MesoscopeCameras` (the loaded dataclass), not
   raw kwargs. Field-to-parameter mapping happens once in `__init__`.
2. **DataLogger ordering.** The DataLogger must be initialized before any VideoSystem so that
   `system_id` registration happens before frame timestamps are emitted.
3. **Output directory.** Always points at the active session's `raw_data` directory derived from
   `SessionData`. Never accept an arbitrary path.

---

## Adding a new camera to Mesoscope-VR

1. **Verify hardware** using `ataraxis@video:camera-setup` (CTI status, list_cameras, runtime requirements).
2. **Test interactively** using the ataraxis video MCP session tools — confirm the camera produces frames
   at the required resolution and framerate.
3. **Add fields to `MesoscopeCameras`** in `sollertia_shared_assets/configuration/mesoscope_configuration.py`
   following the existing `face_*` / `body_*` pattern (`*_camera_index`, `*_camera_quantization`,
   `*_camera_preset`).
4. **Allocate a system ID** from the 50–99 range (next free is 63 at the time of writing).
5. **Extend the binding class** in `sollertia-experiment` to instantiate a third VideoSystem using the new
   fields. Follow the existing face/body lifecycle pattern.
6. **Hand off to the assets plugin's `/system-configuration` skill** to regenerate the host machine's
   system configuration YAML against the new schema. This skill must not call `write_system_configuration_tool`
   or invoke `slsa` directly — system configuration writes are owned by `/system-configuration`.
   The hand-off responsibility includes bumping the `sollertia-shared-assets` version pin so older configurations
   no longer load against the new schema.

---

## Verification checklist

```text
- [ ] New camera fields follow the existing `*_camera_index`, `*_camera_quantization`,
      `*_camera_preset` naming convention in MesoscopeCameras
- [ ] System ID is unique within the 50–99 camera range
- [ ] Binding class accepts MesoscopeCameras as a single argument
- [ ] DataLogger is initialized before VideoSystem instances
- [ ] Handed off to /system-configuration to regenerate the host machine's YAML and bump sollertia-shared-assets
- [ ] Read ataraxis@video:camera-interface for the VideoSystem API
- [ ] Read ataraxis@video:camera-setup for hardware verification
```
