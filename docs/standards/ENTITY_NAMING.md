# Entity Naming Standards

## Overview

Consistent entity naming conventions for Home Assistant, designed to be intuitive, scalable, and resistant to breakdown as the system grows.

---

## Core Principles

1. **Broad to specific** — Start with area, end with the most specific identifier
2. **Underscores throughout** — Consistent with MQTT topic conventions
3. **Function over location** — Name by what it does when purpose is known
4. **Stable identifiers** — Name by the most permanent attribute
5. **Groups + individuals** — Both are addressable; groups for convenience, individuals for granular control

---

## Entity ID Pattern

```
{domain}.{area}_{device_or_function}[_{qualifier}]
```

| Component | Description | Examples |
|-----------|-------------|----------|
| `{domain}` | HA domain (fixed by integration) | `light`, `switch`, `sensor`, `binary_sensor`, `lock` |
| `{area}` | Physical area (lowercase, underscores) | `living_room`, `garage`, `front_porch`, `master_bedroom` |
| `{device_or_function}` | What it is or what it does | `overhead`, `carriage`, `environment`, `outlet` |
| `{qualifier}` | Optional disambiguation | `left`, `temperature`, `floor_lamp`, `west_1` |

---

## Disambiguation Hierarchy

When multiple similar devices exist in an area, disambiguate in this priority order:

| Priority | Method | Use When | Example |
|----------|--------|----------|---------|
| 1 | **Function** | Device has a dedicated purpose | `outlet_floor_lamp` |
| 2 | **Landmark** | Nearby stable reference point | `outlet_by_fireplace` |
| 3 | **Position** | Unique wall/corner/location | `outlet_east_wall` |
| 4 | **Greek letter** | Multiple at same position; general purpose | `outlet_west_alpha`, `outlet_west_beta` |

**Greek alphabet sequence:** alpha, beta, gamma, delta, epsilon, zeta, eta, theta, iota, kappa...

**Rationale:** Naming should reflect the most *stable* identifier. Functions change (lamp moves), landmarks are usually stable (fireplace stays), positions are permanent. Greek letters over numbers convey intentionality — a human made a deliberate choice rather than auto-assignment.

---

## Friendly Names

**Principle:** Omit area from friendly names. Area context is provided by HA's area assignment and inferred by voice assistants.

| Entity ID | Friendly Name | Voice Command (in room) |
|-----------|---------------|------------------------|
| `light.living_room_overhead` | Overhead Light | "Turn on the overhead light" |
| `light.garage_carriage` | Carriage Lights | "Turn on the carriage lights" |
| `sensor.garage_environment_temperature` | Environment Temperature | — |

Cross-room voice commands still work with full context: "Turn on the master bedroom overhead light" (from living room).

---

## Device Type Patterns

### Lights

**Single light:**
```
light.{area}_{function}
light.living_room_overhead
light.front_porch_entry
```

**Multiple lights (same function, grouped):**
```
light.garage_carriage                → group (use for normal control)
light.garage_carriage_left           → individual (granular control)
light.garage_carriage_right          → individual (granular control)
```

**Multiple lights (different functions):**
```
light.living_room_overhead
light.living_room_table_lamp
light.living_room_floor_lamp
```

---

### Switches (including smart outlets)

**Dedicated function:** `switch.living_room_outlet_floor_lamp`

**By landmark:** `switch.living_room_outlet_by_fireplace`

**Fallback (Greek letters):**
```
switch.living_room_outlet_west_alpha
switch.living_room_outlet_west_beta
```

---

### Sensors

**Multi-sensors (environment):**
```
sensor.{area}_environment_temperature
sensor.{area}_environment_humidity
sensor.{area}_environment_illuminance
```

**Motion sensors:**
```
binary_sensor.garage_motion
binary_sensor.garage_motion_driveway     ← if multiple
binary_sensor.garage_motion_interior
```

---

### Contact Sensors (doors/windows)

`_contact` specifies device type:

```
binary_sensor.foyer_entry_door_contact
binary_sensor.master_bedroom_window_east_contact
```

---

### Locks

`lock.` domain already indicates type — no suffix needed:

```
lock.foyer_entry_door
lock.garage_side_door
```

**Door as logical grouping (full example with smart lock):**
```
lock.foyer_entry_door                    → control entity
binary_sensor.foyer_entry_door_lock      → state sensor (locked/unlocked)
binary_sensor.foyer_entry_door_contact   → contact sensor (open/closed)
```

---

### Leak Sensors

```
binary_sensor.basement_leak_water_heater
binary_sensor.kitchen_leak_dishwasher
binary_sensor.laundry_leak_washer
```

---

### Covers (blinds, shades, garage doors)

```
cover.living_room_blinds
cover.living_room_blinds_east
cover.garage_door
```

---

### Cameras

**Pattern:** `camera.{area}_feed_{quality}`

```
camera.driveway_feed_fluent          → Low-res, high-framerate
camera.driveway_feed_clear           → High-res, lower-framerate
```

**Friendly names:** Quality-focused, area omitted: `Feed (Fluent)` / `Feed (Clear)`

**Multiple cameras in same area:** use position or Greek letters:
```
camera.driveway_feed_front_fluent
camera.driveway_feed_alpha_fluent
```

**Related entities:**
```
binary_sensor.driveway_camera_motion
binary_sensor.driveway_camera_person_detected
binary_sensor.driveway_camera_vehicle_detected
```

---

## Groups

Zigbee/Z-Wave groups follow the same pattern as member devices, without qualifier:

```
light.garage_carriage                → group
light.garage_carriage_left           → member
light.garage_carriage_right          → member
```

Groups are the primary control entity for normal use. Individual members remain addressable for granular control.

---

## System Entities

System entities (integrations, HA internals) are **excluded** from these standards. Don't rename them. If Node-RED needs consistent references, create an abstraction layer (mapping) rather than fighting the entity ID.

---

## Floors, Zones & Areas

The `area_id` is the lowercase/underscore form used in entity IDs and MQTT topics.

### Floor 0 — Exterior & Virtual

| Zone | Area Name | Area ID |
|------|-----------|---------|
| Outdoor Living | Front Porch | `front_porch` |
| Outdoor Living | Rear Patio | `rear_patio` |
| Yard Zone | Front Yard | `front_yard` |
| Yard Zone | Driveway | `driveway` |
| Yard Zone | Rear Yard | `rear_yard` |
| Yard Zone | Side Yard | `side_yard` |
| Virtual | Control Room | `control_room` |

### Floor 1 — First Floor

| Area Name | Area ID |
|-----------|---------|
| Dining Room | `dining_room` |
| Foyer | `foyer` |
| Garage | `garage` |
| Half Bathroom | `half_bathroom` |
| Kitchen | `kitchen` |
| Living Room | `living_room` |
| Pantry | `pantry` |
| Rear Hallway | `rear_hallway` |
| Stairway | `stairway` |
| Utility Room | `utility_room` |

### Floor 2 — Second Floor

| Area Name | Area ID | Notes |
|-----------|---------|-------|
| Guest Bathroom | `guest_bathroom` | |
| Guest Room One | `guest_room_one` | |
| Guest Room Two | `guest_room_two` | |
| Hallway | `hallway` | |
| Laundry Room | `laundry_room` | |
| Master Bathroom | `master_bathroom` | |
| Master Bedroom | `master_bedroom` | |
| Master Bedroom Closet One | `master_bedroom_closet_one` | Walk-in |
| Master Bedroom Closet Two | `master_bedroom_closet_two` | Walk-in |
| Office | `office` | |
| Office Bathroom | `office_bathroom` | |

### Floor 3 — Attic

No defined areas yet.

### Area ID Conventions

- Lowercase with underscores
- Singular nouns (`guest_room_one` not `guest_rooms`)
- No abbreviations (`living_room` not `lr`)
- Stable — rename rarely

---

## MQTT Topic Alignment

Entity area and device/function components map directly to MQTT topic hierarchy:

| Entity | Semantic Event Topic |
|--------|---------------------|
| `binary_sensor.garage_motion` | `highland/event/garage/motion_detected` |
| `binary_sensor.basement_leak_water_heater` | `highland/event/basement/leak/water_heater` |
| `lock.foyer_entry_door` | `highland/event/foyer/locked` |

---

## Full Room Example

**Living Room** — overhead light, two table lamps, three outlets, multi-sensor, motion:

```
light.living_room_overhead
light.living_room_table_lamp_east
light.living_room_table_lamp_west

switch.living_room_outlet_floor_lamp
switch.living_room_outlet_west_alpha
switch.living_room_outlet_west_beta

sensor.living_room_environment_temperature
sensor.living_room_environment_humidity
sensor.living_room_environment_illuminance
binary_sensor.living_room_motion
```

---

## Open Questions

- [ ] Climate devices (thermostats, HVAC zones) — Deferred until hardware is selected. Likely ecobee Premium × 2. Pattern TBD.
- [ ] Media players — Not a day-one requirement. Pattern TBD when needed.

---

*Last Updated: 2026-03-26*
